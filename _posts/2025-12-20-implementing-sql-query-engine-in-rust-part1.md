---
layout: "post"
title:  "Implementing SQL Query Engine with Rust"
date:   "2025-12-20 11:23:35 +0530"
categories: database
--- 

# Implementing SQL Query Engine with Rust

To learn about query engines, I've recently been implementing [Query engine](https://github.com/Shourya742/query_engine) from scratch. This part will detail its overall architecture, as well as the implementation of basic SQL `SELECT c1 from t where c2 = 1` queries `select c1, count(c2) max(c2) from t group by c1.`

This part involves:
* catalog
* csv storage
* parser
* binder
* logical planner
* executor
Its milestone is to implement a basic SQL query: `select c1 from t where c2 = 1` to retrieve data from a CSV file.

## catalog

FIrst, we build the most basic component of the database: the catalog. It provides metadata information for the database tables, which is used for the subsequent construction of the binder and logical planner. At the same time, it is a component that runs through the entire query engine processing flow, providing metadata information for the tables.

For simplicity, the concepts of database and schema are not introduced here. The `RootCatalog` directly contains a HashMap to store all `TableCatalogs`. Each `TableCatalog` contains all `ColumnCatalogs`. The definitions are as follows:

```rust
pub struct RootCatalog {
    pub tables: HashMap<TableId, TableCatalog>,
}

pub struct TableCatalog {
    pub id: TableId,
    pub name: String,
    pub column_ids: Vec<ColumnId>,
    pub columns: BTreeMap<ColumnId, ColumnCatalog>
}

pub struct ColumnCatalog {
    pub id:  TableId,
    pub desc: ColumnDesc,
}

pub struct ColumnDesc {
    pub name: String,
    pub data_type: DataType
}
```

## csv_storage

We are building this query engine on Apache Arrow's in-memory data format, it does not provide data file storage and querying itself. Therefore, storage here is an abstraction build on top of Arrow support files, providing a similar iterator method to spawn chunks of data in batches.

The abstract storage trait uses `GAT (generic associated types)`, meaning that the trait internally defines associated types to support different storage, table, and transaction types. It is defined as follows:

```rust
pub trait Storage: Sync + Send + 'static {
    type TableType: Table;

    fn create_table(&self, id: String, filepath: String) -> Result<(), StorageError>;

    fn get_table(&self, id: String) -> Result<Self::TableType, StorageError>;

    fn get_catalog(&self) -> RootCatalog;
}

pub trait Table: Sync + Send + Clone + 'static {
    type TransactionType: Transaction;

    fn read(&self) -> Result<Self::TransactionType, StorageError>;
}

pub trait Transaction: Sync + Send + 'static {
    fn next_batch(&mut self) -> Result<Option<RecordBatch>, StorageError>;
}
```

`CsvStorage` implements the above trait. Each time it is used, it reads a table from storage and then generates a new transaction (containing a new CSV reader) through the read method to allow users to query data in batches.

## parser

The parser layer is used to convert the raw SQL input by the user into an AST, which is then provided to the binder layer and further converted into an expression that is meaningful to the query engine.

The parsing process of raw SQL is similar to that of programming languages, involving lexical and syntactic analysis, which involves a lot of work. Therefore, for the sake of simplicity, the open-source solution sqlparser-rs is used as the parser layer in this implementation.

## binder

Based on the AST provided by the parser, the query engine needs to know a raw string that represents a specific column in the table, as well as the corresponding datatype. Therefore, all this information needs to be converted at the binder layer.

To implement the simplest SQL bind, such as `select c1 from t where c2 = 1`, BoundSelect, the most basic BoundSelect is defined as follows:

```rust
pub struct BoundSelect {
    pub select_list: Vec<BoundExpr>,
    pub from_table: Option<BoundTableRef>,
    pub where_clause: Option<BoundExpr>
}
```

The `Binder` converts the incoming query AST into a BoundSelect. Therefore, to achieve rich SQL syntax support, the first step is to define it explicitly at the binder layer. For example, in the SQL above `c2 = 1`, it represents a type of BoundExpr.

Query engines offer various types of expressions for defining a block within an SQL statement, such as the basic `BoundExpr`:

```rust
pub enum BoundExpr {
    Constant(ScalarValue),
    ColumnRef(BoundColumnRef),
    InputRef(BoundInputRef),
    BinaryOp(BoundBinaryOp),
    TypeCast(BoundTypeCast)
}
```
The above `BoundExpr` can be interpreted as follows:

* Constant: Represents a constant, such as 1, 'a', true, false, null, etc. In `c2 = 1` this context, 1 is a constant expression.
* ColumnRef: Represents a column, for example `c1`, `c2`. It internally includes `ColumnCatalog`.
* InputRef: This is a special expression representing the index of data read from the final `RecordBatch` during the `executor` phase. It converts all `ColumnRefs` to `InputRefs` during `rewriter` phase.
* BinaryOp: Represents a binary operation. For example, the one above `c2 = 1` is a BinaryOp expression with operator `Eq`. Its left and right are also `BoundExpr`.
* TypeCast: Represents a type conversion. For example, in BinaryOp, the left and right sides are of different types, but they can be converted to the same type using implicit cast. In this case, a TypeCast expression can be included when constructing BinaryOp.

## logical planner

At this level, we have all the necessary information for a query and can being constructing the `PlanNode` to initialize the `LogicalPlanNode` and `PhysicalPlanNode`. Example include: Project, Filter and TableScan.

### declarative macro

The initialization of PlanNode heavily utilizes Rust's `as_xxx` property `declarative macro` to generate a large amount of template code. This involves passing a macro name to another macro to achieve macro reuse. `impl_downcast_utility` The macro in the following code expands from the outside in , implementing all the `as_xxx` methods of PlanNode (such as `as_logical_project`). 

```rust
pub trait PlanNode: WithPlanNodeType + PlanTreeNode + Debug + Downcast {
    fn schema(&self) -> Vec<ColumnCatalog> {
        vec![]
    }
}

impl_downcast!(PlanNode);

macro_rules! for_all_plan_nodes {
    ($macro:ident) => {
        $macro! {
            Dummy,
            LogicalTableScan,
            LogicalProject,
            LogicalFilter,
            PhysicalTableScan,
            PhysicalProject,
            PhysicalFilter
        }
    };
}

macro_rules! impl_downcast_utility {
    ($($node_name:ident),*) => {
        impl dyn PlanNode {
            $(
                paste! {
                    #[allow(dead_code)]
                    pub fn [<as_$node_name:snake>](&self) -> self::result::Result<&$node_name, ()> {
                        self.downcast_ref::<$node_name>().ok_or_else(|| ())
                    }
                }
            )*
        }
    }
}

for_all_plan_nodes! { impl_downcast_utility }
```

### rewriter

Once we have defined `LogicalPlanNode` and `PhysicalPlanNode`, we need to build them and assemble them into a PlanTree to represent the entire query plan and provide execution for the executor.

A PlanTree is constructed by assembling PlanNodes. Except for the bottom-level TableScan, each PlanNode contains children to represent its lower-level PlanNodes. The trait is defined as follows:

```rust
pub trait PlanTreeNode {
    /// Get the child plan nodes.
    fn children(&self) ->  Vec<PlanRef>;

    /// CLone the node with new children for rewriting the plan node.
    fn clone_with_children(&self, children: Vec<PlanRef>) -> PlanRef;
}
```

Constructing a LogicalPlanNode is relatively simple. You just need to manually assemble the information in BoundSelect into a LogicalPlanTree. Note that a PlanTree is represented as a PlanRef in the code, so `Arc<dyn PlanNode>` some downcasts are need to obtain the specific PlanNode.

When constructing a PhysicalPlanNode, a mechanism is introduced `visitor pattern` to implement PlanNode rewriter and BoundExpr rewriter.

First, let's discuss the PlanNode rewriter, as defined below:

```rust
macro_rules! def_rewriter {
    ($($node_name:ident),*) => {
        pub trait PlanRewriter {
            paste! {
                fn rewrite(&mut self, plan: PlanRef) -> PlanRef {
                    match plan.node_type() {
                        $(
                            PlanNodeType::$node_name => self.[<rewrite_$node_name:snake>](plan.downcast_ref::<$node_name>().unwrap()),
                        )*
                    }
                }

                $(
                    fn [<rewrite_$node_name:snake>](&mut self, plan: &$node_name) -> PlanRef {
                        let new_children = plan.children().into_iter().map(|child| self.rewrite(child.clone())).collect_vec();
                        plan.clone_with_children(new_children)
                    }
                )*
            }
        }
    };
}

for_all_plan_nodes! { def_rewriter }
```

let see the expanded code:

```rust
pub trait PlanRewriter {
    fn rewrite(&mut self, plan: PlanRef) -> PlanRef {
        match plan.node_type() {
            PlanNodeType::Dummy => {
                self.rewrite_dummy(plan.downcast_ref::<Dummy>().unwrap())
            },
            PlanNodeType::LogicalTableScan => {
                self.rewrite_logical_table_scan(plan.downcast_ref::<LogicalTableScan>())
            },
            PlanNodeType::LogicalProject => {
                self.rewrite_logical_project(plan.downcast_ref::<LogicalProject>())
            },
            PlanNodeType::LogicalFilter => {
                self.rewrite_logical_filter(plan.downcast_ref::<LogicalFilter>())
            },
            PlanNodeType::PhysicalTableScan => {
                self.rewrite_physical_table_scan(plan.downcast_ref::<PhysicalTableScan>())
            },
            PlanNodeType::PhysicalProject => {
                self.rewrite_physical_project(plan.downcast_ref::<PhysicalProject>())
            },
            PlanNodeType::PhysicalFilter => {
                self.rewrite_physical_filter(plan.downcast_ref::<PhysicalFilter>())
            }
        }
    }

    fn rewrite_dummy(&mut self, plan: &Dummy) -> PlanRef {
        //...
    }

    fn rewrite_logical_table_scan(&mut self, plan: &LogicalTableScan) -> PlanRef {
        let new_children = plan.children().into_iter().map(|child| self.rewrite(child.clone())).collect_vec();
        plan.clone_with_children(new_children)
    }

    fn rewrite_logical_project(&mut self, plan: &LogicalProject) -> PlanRef {
        // ...
    }

    fn rewrite_logical_filter(&mut self, plan: &LogicalFilter) -> PlanRef {
        // ...
    }

    fn rewrite_physical_table_scan(&mut self, plan: &PhysicalTableScan) -> PlanRef {
        // ...
    }

    fn rewrite_physical_project(&mut self, plan: &PhysicalProject) -> PlanRef {
        // ...
    }

    fn rewrite_physical_filter(&mut self, plan: &PhysicalFilter) -> PlanRef {
        // ...
    }
}
```

This trait contains some default methods to rewrite all PlanNodes. The rewrite method for each PlanNode is the same: first, the rewrite method of all children is executed to obtain new_children, and then clone_with_children is called to generate a new PlanNode.

This approach provides a `unified code framework`, allowing rewriting of both LogicalPlanTree and PhysicalPlanTree using PlanRewriter. If custom rewrite logic is needed to implement specific functionalities, the corresponding methods of PlanRewriter can be overridden, such as PhysicalRewriter and inputRefRewriter.

* PhysicalRewriter: Only overrides the rewrite_logical_xxx method to convert a LogicalPlanTree into a PhysicalPlanTree.
* InputRefRewriter: Only overrides the rewrite_logical_xxx method, rewriting the ColumnRef expression as InputRef from the bottom up.

In addition to PlanNode rewriter, there is also BoundExpr rewriter, defined as follows:

```rust
pub trait ExprRewriter {
    fn rewrite_expr(&self, expr: &mut BoundExpr) {
        match expr {
            BoundExpr::Constant(_) => self.rewrite_constant(expr),
            BoundExpr::ColumnRef(_) => self.rewrite_column_ref(expr),
            BoundExpr::InputRef(_) => self.rewrite_input_ref(expr),
            BoundExpr::BinaryOp(_) => self.rewrite_binary_op(expr),
            BoundExpr::TypeCast(_) => self.rewrite_type_cast(expr),
        }
    }

    fn rewrite_constant(&self, _: &mut BoundExpr) {}

    fn rewrite_column_ref(&self, _: &mut BoundExpr) {}

    fn rewrite_input_ref(&self, _: &mut BoundExpr) {}

    fn rewrite_type_cast(&self, _: &mut BoundExpr) {}

    fn rewrite_binary_op(&self, _: &mut BoundExpr) {
        match expr {
            BoundExpr::BinaryOp(e) => {
                self.rewrite_expr(&mut e.left);
                self.rewrite_expr(&mut e.right);
            }
            _ => unreachable!()
        }
    }
}
```

For example, InputRefRewriter needs to implement ExprRewriter and override the rewrite_column_ref method to convert ColumnRef to InputRef when visiting LogicalPlanTree.

At this point, we have obtained a PhysicalPlanTree. The next step is to determine how the executor constructs the Volcano execution model.

## Executor

The core of the execution engine is a Vectorized Volcano Model. Therefore, we still need a visitor pattern to traverse the PlanTree. A similar PlanVisitor implementation, modeled after the PlanRewriter above, is provided:

```rust
macro_rules! def_rewriter {
    ($($node_name:ident), *) => {
        pub trait PlanVisitor<R> {
            paste! {
                fn visit(&mut self, plan: PlanRef) -> Option<R> {
                    match plan.node_type() {
                        $(
                            PlanNodeType::$node_name => self.[<visit_$node_name:snake>](plan.downcast_ref::<$node_name>().unwrap()),
                        )*
                    }
                }

                $(
                    fn [<visit_$node_name:snake>](&mut self, _plan: &$node_name) -> Option<R> {
                        unimplemented!("the {} is not implemented visitor yet", stringify!($node_name))
                    }
                )*
            }
        }
    }
}

for_all_plan_nodes! { def_rewriter }
```

The entry point for constructing an executor is ExecutorBuilder, which implements PlanVisitor. It converts the input PhysicalPlanTree into a DAG assembled by Executors, and then executes it.

For the SQL we want to implement, `select c1 from t where c2 = 1` we only need three executors: project, filter and table_scan. The technologies used will be introduced below.

### future-async-stream

First, for an execution operator, it is essentially an iterator. You can retrieve data using `next_batch`, but  for clearer code logic, the execution logic is wrapped in `future-async-stream`. From the outermost abstraction, all operators are assembled into a stream, extracting data sequentially from the top layer of the DAG downwards, passing through the execution logic of various operators, and finally returning to the top layer.

Starting with TableScanExecutor, which is the lowest-level executor, it takes storage as input and retrieves data through transaction.next_batch(). Therefore, its core logic is as follows:

```rust
#[try_stream(boxed, ok = RecordBatch, error = ExecutorError)]
pub async fn execute(self) {
    let table_id = self.plan.logical().table_id();
    let table = self.storage.get_table(table_id)?;
    let mut tx = table.read()?;
    loop {
        match tx.next_batch() {
            Ok(batch) => {
                if let Some(batch) = batch {
                    yield batch;
                } else {
                    break;
                }
            }
            Err(err) => return Err(err.into())
        }
    }
}
```

### Evaluator

With TableScanExecutor, to select a specific column of data, ProjectExecutor needs to be implemented. It inputs the RecordBatch from downstream into a BoundExpr for calculation. THe following are some types of BoundExpr:

* InputRef: Retrieves a specific column of data from RecordBatch.
* Constant: Construct a constant column
* TypeCast: Converts column data to a specified type.
* BinaryOp: Calculates The BinaryOperator for left and right expr, such as: `left + right`, `left = right` etc.

For SQL: `select c1 from t where c2 = 1` Project Executor only needs to use `eval InputRef` to retrieve the column for the index, and then construct a new `RecordBatch` and return it. See the code below:

```rust
#[try_stream(boxed, ok = RecordBatch, error = ExecutorError)]
pub async fn execute(self) {
    #[for_await]
    for batch in self.child {
        let batch = batch?;
        let columns = self.exprs.iter().map(|e| e.eval_column(&batch)).try_collect();
        let fields = self.exprs.iter().map(|e| e.eval_fields(&batch)).collect();
        let schema = SchemaRef::new(Schema::new_with_metadata(fields, batch.schema().metadata().clone()));
        yield  RecordBatch::try_new(schema, columns?)?;
    }
}
```

### array_compute

This step requires processing `select c1 from t where c2 = 1` the WHERE expression in the SQL statement. Therefore, the `compute` module of `arrow` is introduced, which provides many calculation methods between arrays, such as `add`, `multiply` and `eq`.

For example, the calculation of binary_op used in the where expression is shown in the following code:

```rust
pub fn binary_op(left: &ArrayRef, right: &ArrayRef, op: &BinaryOperator) -> Result<ArraryRef, ExecutorError> {
    match op {
        BinaryOperator::Plus => arithmetic_op!(left, right, add),
        BinaryOperator::Minus => arithmetic_op!(left, right, subtract),
        BinaryOperator::Multiply => arithmetic_op!(left, right, multiply),
        BinaryOperator::Divide => arithmetic_op!(left, right, divide),
        BinaryOperator::Gt => Ok(Arc::new(gt_dyn(left, right)?)),
        BinaryOperator::Lt => Ok(Arc::new(lt_dyn(left, right)?)),
        BinaryOperator::GtEq => Ok(Arc::new(gt_eq_dyn(left, right)?)),
        BinaryOperator::LtEq => Ok(Arc::new(lt_eq_dyn(left, right)?)),
        BinaryOperator::Eq => Ok(Arc::new(eq_dyn(left, right)?))
        _ => todo!()
    }
}
```

Additionally, arrow provides the filter_record_batch method, which allows us to filter out matching rows and organise them into a new RecordBatch simply by passing in a boolean array as the predicate.

## Summary

At this point `select c1 from t where c2 = 1` our query engine can retrieve the data from this SQL statement. Meanwhile, the modules of a simple query engine have been initialized. Subsequently, the corresponding features, such as aggregation and join, can be added directly to specific modules.