# 查询优化器

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