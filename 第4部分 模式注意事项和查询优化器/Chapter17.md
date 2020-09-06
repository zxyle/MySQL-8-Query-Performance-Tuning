# 查询优化器

当您向MySQL提交查询以执行时，它不只是读取数据并返回数据那样简单。的确，对于从单个表中请求所有数据的简单查询，如何检索数据没有太多选择。但是，大多数查询更为复杂（有些更为复杂），并且完全按照编写的方式执行查询绝对不是获得结果的最有效方法。阅读索引时，您已经谈到了这种复杂性。您可以添加索引的选择，连接顺序，用于执行连接的算法，各种连接优化等。那就是优化器起作用的地方。

优化器的主要工作是准备要执行的查询并确定最佳查询计划。第一阶段涉及对查询进行转换，目的是可以以比原始查询更低的成本执行重写的查询。第二阶段包括计算执行查询的各种方式的成本，并确定最便宜的选项。

## Transformations

## Cost-Based Optimization

### The Basics: Single Table SELECT

### Table Join Order

### Default Filtering Effects

### The Query Cost

## Join Algorithms

### Nested Loop

### Block Nested Loop

### Hash Join

## Join Optimizations

### Index Merge

#### Intersection Algorithm

#### Union Algorithm

#### Sort-Union Algorithm

#### Performance Considerations

#### Configuration

### Multi-Range Read (MRR)

### Batched Key Access (BKA)

### Other Optimizations

#### Condition Filtering

#### Derived Merge

#### Engine Condition Pushdown

#### Index Condition Pushdown

#### Index Extensions

#### Index Visibility

#### Loose Index Scan

#### Range Access Method

#### Semijoin

#### Skip Scan

#### Subquery Materialization

## Configuring the Optimizer

### Engine Costs

### Server Costs

### Optimizer Switches

### Optimizer Hints

### Index Hints

### Configuration Options

## Resource Groups

### Retrieving Information About Resource Groups

### Managing Resource Groups

### Assigning Resource Groups

### Performance Considerations

## Summary