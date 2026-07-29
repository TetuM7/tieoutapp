# Statement Engine — Corpus Baseline Scorecard

Driven by `extractStatement()` over the annotated corpus. Regenerate: `pnpm --filter @tieout/statement-engine eval`.
Rates are % of fixtures. **falseRecon** (claimed reconciled to a wrong closing balance) must be 0.

| bank            |   n | route | inst | last4 | perStart | perEnd | endBal | tie-out | computed | falseRecon |
|-----------------|----:|------:|-----:|------:|---------:|-------:|-------:|--------:|---------:|-----------:|
| jpmorgan-chase  |  31 |  100% |  100% |  100% |  100%    |  100%  |   97% |   87%   |   81%    |          0 |
| bank-of-america |  38 |   97% |   97% |   92% |   95%    |   95%  |   92% |   76%   |   82%    |          0 |
| wells-fargo     |  37 |  100% |  100% |  100% |  100%    |   97%  |   97% |   73%   |   54%    |          0 |
| citibank        |  63 |  100% |   98% |   98% |   98%    |   97%  |   98% |   70%   |   68%    |          0 |
| capital-one     |  70 |  100% |  100% |   94% |  100%    |  100%  |   97% |   83%   |   76%    |          0 |
| us-bank         |  42 |  100% |  100% |   95% |   98%    |   98%  |   95% |   88%   |   83%    |          0 |
|                 |     |       |      |       |          |        |        |         |          |            |
| ALL             | 281 |  100% |   99% |   96% |   99%    |   98%  |   96% |   79%   |   74%    |          0 |

## Row provenance (D-030)

Itemized rows stamped with a PDF page + bbox (`source.bbox`), by matching each row back to its PDF line. Only counts fixtures whose register was itemized into rows.

| bank            | rows | sourced | coverage |
|-----------------|-----:|--------:|---------:|
| jpmorgan-chase  |  446 |     313 |   70% |
| bank-of-america |  384 |     241 |   63% |
| wells-fargo     |  494 |     427 |   86% |
| citibank        |  233 |     229 |   98% |
| capital-one     | 2496 |    2379 |   95% |
| us-bank         |  636 |     558 |   88% |
| ALL             | 4689 |    4147 |   88% |

