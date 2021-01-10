# rdb-miscs

这个项目主要记录关系型数据库实践中遇到的问题。我会尝试在某个合适的数据集上重现这些问题，让感兴趣的人可以复现、理解个中缘由，最终能付诸实践。

## Datasets

* [imdb](./datasets/imdb.md)
* [crunchbase 2013 snapshot](./datasets/crunchbase_2013.md)

## Questions and Answers

* [为什么不应该使用 OFFSET](./qa/offset.md)
* [特定场景下应对写偏斜 (write skew) 的一种方案](./qa/write-skew.md)
* [什么时候宁可全表扫描也不走索引](./qa/prefer-full-table-scan.md)
* [为什么 SELECT COUNT(\*) 很慢](./qa/count_star.md)
* [通过 Demo 理解一致性模型](./qa/consistency-models-via-demos.md)

## Readings

* [The Unofficial MySQL 8.0 Optimizer Guide](http://www.unofficialmysqlguide.com/)
* [A Critique of ANSI SQL Isolation Levels](https://arxiv.org/pdf/cs/0701157.pdf)
