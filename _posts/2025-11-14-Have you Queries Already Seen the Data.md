---
layout: post
title: Have your Queries Already Seen the Data? Data-Privilege in Tabular Benchmarks
date: 2025-11-14
description: We take a look at how many popular benchmarks for evaluating natural language interfaces to tabular data
  contain "data-privileged" queries—questions that reference specific components of the data that real users wouldn't
  know about in open-domain settings.
tags: trl open-domain benchmarks
categories: article
pretty_table: true
toc:
  sidebar: left
---

> **Note:** This blog post is a deep-dive into a critical issue surfaced in our recent paper _"Are We Asking the Right
> Questions? On Ambiguity in Natural Language Queries for Tabular Data Analysis"_ accepted to the
> [AI for Tabular Data Workshop at EurIPS 2025](https://sites.google.com/view/eurips25-ai-td/home).

> **TL;DR:** Many popular benchmarks for evaluating natural language interfaces to tabular data contain
> "data-privileged" queries—questions that reference specific database structures, internal codes, or data containers
> that real users wouldn't know about in open-domain settings. Our analysis of 15 datasets reveals that up to 70% of
> queries in complex analysis benchmarks and 26-27% in widely-used text-to-SQL datasets contain such privileged
> information, fundamentally undermining evaluations by providing unrealistic shortcuts. To properly test open-domain
> capabilities, we need to either carefully adapt existing datasets to remove privileged information while maintaining
> realistic data scope specification, or shift to query-first construction methodologies that mirror how real users
> formulate information needs.

## When Your Test Data Knows Too Much

Natural language interfaces to databases have come a long way. What started as systems that let users query a single,
known database with questions like "How many employees work in the sales department?" are evolving into something far
more ambitious: open-domain systems that identify relevant tabular data from vast, unknown corpora before answering
queries. This shift is happening across the board—from text-to-SQL generation[^1][^2] to question
answering[^7][^16][^9] to full-scale data analysis[^6][^3].

But there's a problem hiding in plain sight. As researchers start evaluating open-domain systems they plainly adapt
existing datasets from closed-domain settings (where users know exactly which database they're querying) or device new
specific datasets. While users formulate queries over an unknown corpus of tabular data, going from query to data, these
datasets are constructed the other way around, starting out with (a) specific table(s) and deriving queries from this.
If not explicitly addressed, such dataset construction methods inadvertently introduce an unrealistic advantage into the
data: many queries in these benchmarks contain privileged information that real users would never have access to.

Consider this query from the HiTab dataset: _"What was the number of the ashrs of polish refugees?"_[^10] The term "
ashrs" isn't a word you'd find in a dictionary, it's a column header copied directly from a specific table. Or this one
from DA-Eval: _"Check if the RHO_OLD column follows a normal distribution."_[^11] A real user asking questions about
data they haven't seen wouldn't know that a column called "RHO_OLD" exists, let alone reference it by its exact database
identifier.

These aren't isolated examples. In our analysis of 15 popular datasets spanning question answering, text-to-SQL, and
data analysis tasks, we found that some benchmarks have up to 70% of their queries containing such privileged
information. This matters because when we evaluate systems on these queries, we're not testing their ability to work in
realistic open-domain settings, instead we're testing them on a fundamentally easier task where users magically know the
underlying data structure, directly tying a query to specific data.

## What Is Data-Privilege, and Why Should We Care?

At its core, a data-privileged query is one that betrays knowledge the user shouldn't have. In a true open-domain
setting, users approach the system with information needs grounded in their understanding of the world and the system
they are interacting with [^8], not in knowledge of how some specific dataset happens to be structured. When a user asks
about "quarterly revenue trends for technology companies," they're expressing a natural information need. When they ask
about "the avg_revenue_q1 column for rows where industry_code='TECH'," they're revealing that they've already seen the
data.

This distinction matters profoundly for evaluation. _Data-privileged queries_ provide an unrealistic signal that makes
the task appear simpler than it actually is. They create a shortcut: instead of having to understand the user's
conceptual query and map it to whatever data structures might be relevant in a massive corpus, the system can often
pattern-match directly to the referenced structural elements. This fundamentally undermines what we claim to be testing
in open-domain scenarios.

We identify three distinct manifestations of data-privilege in queries:

#### 1. Structural References - When Queries Speak Database

Structural references occur when queries use terminology that sounds more like database schema than natural
language—phrases that feel "copied" rather than "composed." The most obvious cases involve programming conventions:
queries asking about "SalePrice" (camelCase) or "gdpPercap_1982" (underscores and suffixes) are clearly referencing
specific field names. But the pattern can be subtler: asking for "Aaron Doran's potential score" rather than his "skill
level" or "rating" suggests familiarity with how a sports database labels its columns[^5].

Database-specific concepts provide another giveaway. Queries that ask for "the record with index 5" or "the primary key"
use language from data management, not everyday information seeking. These concepts are highly indicative of directly
interacting with a database instead of leaving this task to the system. Consider this query from TableBench: _"What is
the correlation between a country's 'carbon dioxide emissions per year (tons per person)' and its 'average emission per
km² of land'?"_[^19]. Those precise metric definitions in quotes strongly suggest knowledge of exactly how these
measurements are labeled in a specific dataset.

#### 2. Value References - Knowing the Database Contents

Value references reveal knowledge of what specific data values exist in underlying tables. The clearest cases involve
internal identifiers, i.e., codes or keys that exist purely within a dataset's organizing logic. Consider this from
MMQA: _"Which clubs located in 'AKW' have members holding either 'President' or 'Vice-President' positions?"_[^18]
Unless "AKW" is publicly known, this three-letter code suggests the user has looked at the table. Similarly, queries
about _"authors who publish books in both 'MM' and 'LT' series"_[^4] or _"clients whose complaint type is 'TT'"_[^5] use
cryptic codes that point to internal categorization schemes.

Not all value references indicate leakage of privileged information. Publicly knowable named entities like people,
places, organizations, or dates don't necessarily indicate data-privilege. Asking about "Janja Garnbret" or "the 2024
Olympics" uses world knowledge, not dataset-specific knowledge. The distinction lies in whether the specificity comes
from general knowledge or from having seen the particular data.

#### 3. Container References - Breaking the Fourth Wall

Container references explicitly acknowledge working with a data artifact, breaking the illusion of asking about the
world. Phrases like "in the dataset," "according to the table," or "using the provided spreadsheet" directly reference
the data container. Consider the query _"Load the data into the SQLite database"_ from DA-Code [^12], this isn't
maintaining any fiction of open-domain interaction. Even subtle conceptual references like "using the provided dataset,
find the top five most frequent qualifications" assume a bounded artifact that has been "provided" rather than
discovered.

## Data-Privilege Across 15 Popular Benchmarks

To get a grasp on the prevalence of data-privileged queries, we systematically analyzed 15 datasets commonly used to
evaluate natural language interfaces to tabular data, spanning single-table question answering, multi-table reasoning,
text-to-SQL, and data analysis tasks (see Table 1). Using LLM-based classifiers validated against expert annotations, we
labeled queries for structural references, value references, and container references. Figure 1 presents the
distribution of data-privileged queries across these datasets.

| **Dataset**             |     **Task**     | **Open-Domain** | **#Queries** |
| ----------------------- | :--------------: | :-------------: | -----------: |
| WikiTableQuestions[^17] | Single Table QA  |       ❌        |       14,151 |
| TabMWP[^15]             | Single Table QA  |       ✅        |       38,901 |
| CRT-QA[^20]             | Single Table QA  |       ✅        |          728 |
| HiTab[^10]              | Single Table QA  |       ✅        |       10,672 |
| OpenWikiTables[^13]     | Single Table QA  |       ✅        |       67,023 |
| OTT-QA[^9]              | Single Table QA  |       ✅        |        4,372 |
| FeTaQA[^16]             | Single Table QA  |       ✅        |       10,330 |
| TableBench[^19]         | Single Table QA  |       ❌        |          886 |
| QTSumm[^21]             |  Multi Table QA  |       ❌        |       10,440 |
| MMQA[^18]               |  Multi Table QA  |       ✅        |        3,313 |
| Spider[^4]              |   Text-to-SQL    |       ❌        |       11,840 |
| BIRD[^5]                |   Text-to-SQL    |       ❌        |       10,962 |
| DA-Code[^12]            | Tabular Analysis |       ❌        |          500 |
| KramaBench[^14]         | Tabular Analysis |        ~        |          104 |
| DA-Eval[^11]            | Tabular Analysis |       ❌        |          257 |

**Table 1:** Overview of analyzed datasets and their characteristics.

The most striking pattern is the correlation between task complexity and data-privilege rates. Complex tabular analysis
benchmarks show dramatically higher data-privilege: DA-Eval [^11] at 70% (dominated by structural references at 63%) and
DA-Code at 59% (primarily container references)[^12]. In contrast, some simpler question-answering datasets achieve much
better data-independence, with FeTaQA [^16] at just 0.4%, OTT-QA [^9] at 5.4%, and HiTab[^10] at 9.6%. This makes
intuitive sense, specifying complex analytical operations without referencing specific data structures is genuinely
harder. We cannot properly assess systems' abilities to translate complex insight needs into appropriate analyses if our
test queries already encode privileged knowledge about data organization.

{% include figure.liquid path="assets/img/blog/data_privilege_by_dataset.png" class="img-fluid rounded z-depth-1"
zoomable=true %}
**Figure 1:** Distribution of data-privileged queries across 15 tabular benchmarks, broken down by reference type.

A critical finding for the community concerns text-to-SQL datasets. Spider[^4] and BIRD [^5], while originally designed
for closed-domain scenarios, are increasingly being repurposed to evaluate open-domain text-to-SQL systems[^1][^2]. Yet
our analysis reveals that 27% and 26% of their queries respectively contain data-privileged information[^4][^5]. While
these rates are more moderate than some of the complex analysis benchmarks, they represent hundreds of queries that
provide unrealistic shortcuts in open-domain settings. Researchers adapting these datasets should be aware that a
substantial fraction of queries assume knowledge users wouldn't have when querying unknown databases. Even among
datasets explicitly intended for open-domain evaluation, design choices matter: OpenWikiTables, despite its open-domain
goals, shows 32% data-privilege[^13], likely because their methodology for decontextualization with language models
insufficiently accounts for the leakage of privileged information.

## Beyond Data-Privilege - Aligning Queries with Users' Mental Model

Data-privileged queries are actually just one manifestation the broader challenge of aligning queries with users' mental
models when interacting with open-domain systems[^8]. While queries should be data-independent, the queries only make
sense if they specify a data scope, that is, when they provide information on what sort of data a query is
targeting [^22]. In a closed-domain setting, the data context itself provides contextual scaffolding that grounds the
data boundaries of a query.

Consider this query from Spider: _"What clubs are there?"_[^4] No structural references, no value references, no
container references—yet it's nonsensical in open-domain. What clubs? Where? A real user would specify the data scope,
for instance to "What soccer clubs are there in Manchester?". The query only works because Spider provides a closed
database where "clubs" has unambiguous meaning.

Closed-domain settings provide enormous implicit information. The database context itself bounds the scope and resolves
ambiguities. Open-domain systems eliminate this scaffolding. Users must either explicitly specify data scope or rely on
systems to make reasonable inferences, a fundamentally different kind of completeness than closed-domain queries
require. For a detailed framework around this division of labor between users and systems in grounding queries,
see [our full paper](https://arxiv.org/abs/2511.04584).

## Looking Forward: Toward Realistic Open-Domain Evaluation

The prevalence of data-privilege across popular benchmarks demands that we adapt existing datasets and fundamentally
rethinking how we construct new ones, when we want to evaluate open-domain systems.

If we want to use existing datasets, we should adapt them thoroughly to the open-domain setting. Problematic queries
should be adapted by removing privileged information while ensuring they retain sufficient data scope specification to
enable realistic retrieval. This requires careful manual review to maintain the query's intent while aligning it with
users' natural mental models. For datasets where such adaptation isn't feasible, researchers should at minimum document
data-privilege rates and consider whether the benchmark appropriately tests their evaluation objectives.

An interesting direction for constructing new datasets is to shift the paradigm from data-first to query-first
methodologies. Current practices—starting with tables and deriving queries from them—has a high likelihood of leaking
structural knowledge into the queries themselves. Instead, we should investigate collection approaches that mirror
realistic user workflows, gathering information needs from domain experts or users who haven't seen the underlying data,
then identifying relevant tables to satisfy those needs. This inverts the construction process to match how open-domain
systems actually operate. Such methodologies are more resource-intensive but essential for benchmarks that are truly
aligned with realistic use, truly testing open-domain capabilities.

Most critically, the research community must develop awareness of what our benchmarks actually measure. When adapting
closed-domain datasets for open-domain evaluation, we cannot simply assume the queries transfer appropriately. When
claiming to evaluate table retrieval or open-domain analysis capabilities, we must verify that our test queries don't
provide unrealistic shortcuts that bypass the very capabilities we're trying to assess. The queries we use fundamentally
shape what capabilities systems develop. Aligning systems with many of our current benchmarks may be teaching them to
exploit privileged information rather than to genuinely understand user information needs and discover relevant data.

---

If you found this blog post interesting, please check out our full
paper: [Are We Asking the Right Questions? On Ambiguity in Natural Language Queries for Tabular Data](https://arxiv.org/abs/2511.04584)

Cite as:

```
@inproceedings{gommAreWeAsking2025,
  title = {Are {{We Asking}} the {{Right Questions}}? {{On Ambiguity}} in {{Natural Language Queries}} for {{Tabular Data Analysis}}},
  shorttitle = {Are {{We Asking}} the {{Right Questions}}?},
  booktitle = {{{AI}} for {{Tabular Data}} Workshop at {{EurIPS}} 2025},
  author = {Gomm, Daniel and Wolff, Cornelius and Hulsebos, Madelon},
  year = 2025,
  url = {https://arxiv.org/abs/2511.04584}
}

```

---

## References

[^1]:
    X. Zhang, D. Wang, L. Dou, Q. Zhu, and W. Che, "MURRE: Multi-Hop Table Retrieval with Removal for Open-Domain
    Text-to-SQL," in _Proceedings of the 31st International Conference on Computational Linguistics_, O. Rambow, L. Wanner,
    M. Apidianaki, H. Al-Khalifa, B. D. Eugenio, and S. Schockaert, Eds., Abu Dhabi, UAE: Association for Computational
    Linguistics, Jan. 2025, pp. 5789–5806.

[^2]:
    M. Kothyari, D. Dhingra, S. Sarawagi, and S. Chakrabarti, "CRUSH4SQL: Collective Retrieval Using Schema
    Hallucination For Text2SQL," in _Proceedings of the 2023 Conference on Empirical Methods in Natural Language
    Processing_, H. Bouamor, J. Pino, and K. Bali, Eds., Singapore: Association for Computational Linguistics, Dec. 2023,
    pp. 14054–14066. doi: [10.18653/v1/2023.emnlp-main.868](https://doi.org/10.18653/v1/2023.emnlp-main.868).

[^3]:
    K. Kong _et al._, "OpenTab: Advancing Large Language Models as Open-domain Table Reasoners," presented at the The
    Twelfth International Conference on Learning Representations, Oct. 2023.

[^4]:
    T. Yu _et al._, "Spider: A Large-Scale Human-Labeled Dataset for Complex and Cross-Domain Semantic Parsing and
    Text-to-SQL Task," Feb. 02, 2019, _arXiv_: arXiv:1809.08887.
    doi: [10.48550/arXiv.1809.08887](https://doi.org/10.48550/arXiv.1809.08887).

[^5]:
    J. Li _et al._, "Can LLM Already Serve as A Database Interface? A BIg Bench for Large-Scale Database Grounded
    Text-to-SQLs," in _Advances in Neural Information Processing Systems_, 2024.

[^6]:
    J. Wang and G. Li, "AOP: Automated and Interactive LLM Pipeline Orchestration for Answering Complex Queries,"
    presented at the Conference on Innovative Database Research, 2025.

[^7]:
    J. Herzig, T. Müller, S. Krichene, and J. Eisenschlos, "Open Domain Question Answering over Tables via Dense
    Retrieval," in _Proceedings of the 2021 Conference of the North American Chapter of the Association for Computational
    Linguistics: Human Language Technologies_, K. Toutanova _et al._, Eds., Online: Association for Computational
    Linguistics, Jun. 2021, pp. 512–519.
    doi: [10.18653/v1/2021.naacl-main.43](https://doi.org/10.18653/v1/2021.naacl-main.43).

[^8]: D. A. Norman, "Some Observations on Mental Models," in _Mental Models_, Psychology Press, 1983.
[^9]:
    W. Chen, M.-W. Chang, E. Schlinger, W. Y. Wang, and W. W. Cohen, "Open Question Answering over Tables and Text,"
    presented at the International Conference on Learning Representations, Oct. 2020.

[^10]:
    Z. Cheng _et al._, "HiTab: A Hierarchical Table Dataset for Question Answering and Natural Language Generation,"
    in _Proceedings of the 60th Annual Meeting of the Association for Computational Linguistics (Volume 1: Long Papers)_, S.
    Muresan, P. Nakov, and A. Villavicencio, Eds., Dublin, Ireland: Association for Computational Linguistics, May 2022, pp.
    1094–1110. doi: [10.18653/v1/2022.acl-long.78](https://doi.org/10.18653/v1/2022.acl-long.78).

[^11]:
    X. Hu _et al._, "InfiAgent-DABench: Evaluating Agents on Data Analysis Tasks," in _Proceedings of the 41st
    International Conference on Machine Learning_, PMLR, Jul. 2024, pp. 19544–19572.

[^12]:
    Y. Huang _et al._, "DA-Code: Agent Data Science Code Generation Benchmark for Large Language Models," in
    _Proceedings of the 2024 Conference on Empirical Methods in Natural Language Processing_, Association for Computational
    Linguistics, Oct. 2024. doi: [10.18653/v1/2024.emnlp-main.748](https://doi.org/10.18653/v1/2024.emnlp-main.748).

[^13]:
    S. Kweon, Y. Kwon, S. Cho, Y. Jo, and E. Choi, "Open-WikiTable : Dataset for Open Domain Question Answering with
    Complex Reasoning over Table," in _Findings of the Association for Computational Linguistics: ACL 2023_, A. Rogers, J.
    Boyd-Graber, and N. Okazaki, Eds., Toronto, Canada: Association for Computational Linguistics, Jul. 2023, pp. 8285–8297.
    doi: [10.18653/v1/2023.findings-acl.526](https://doi.org/10.18653/v1/2023.findings-acl.526).

[^14]:
    E. Lai _et al._, "KramaBench: A Benchmark for AI Systems on Data-to-Insight Pipelines over Data Lakes," Jun. 06,
    2025, _arXiv_: arXiv:2506.06541. doi: [10.48550/arXiv.2506.06541](https://doi.org/10.48550/arXiv.2506.06541).

[^15]:
    P. Lu _et al._, "Dynamic Prompt Learning via Policy Gradient for Semi-structured Mathematical Reasoning,"
    presented at the The Eleventh International Conference on Learning Representations, Sep. 2022.

[^16]:
    L. Nan _et al._, "FeTaQA: Free-form Table Question Answering," _Transactions of the Association for Computational
    Linguistics_, vol. 10, pp. 35–49, 2022, doi: [10.1162/tacl_a_00446](https://doi.org/10.1162/tacl_a_00446).

[^17]:
    P. Pasupat and P. Liang, "Compositional Semantic Parsing on Semi-Structured Tables," in _Proceedings of the 53rd
    Annual Meeting of the Association for Computational Linguistics and the 7th International Joint Conference on Natural
    Language Processing (Volume 1: Long Papers)_, C. Zong and M. Strube, Eds., Beijing, China: Association for Computational
    Linguistics, Jul. 2015, pp. 1470–1480. doi: [10.3115/v1/P15-1142](https://doi.org/10.3115/v1/P15-1142).

[^18]:
    J. Wu, L. Yang, D. Li, Y. Ji, M. Okumura, and Y. Zhang, "MMQA: Evaluating LLMs with Multi-Table Multi-Hop Complex
    Questions," presented at the The Thirteenth International Conference on Learning Representations, Oct. 2024.

[^19]:
    X. Wu _et al._, "TableBench: A Comprehensive and Complex Benchmark for Table Question Answering," Mar. 18, 2025,
    _arXiv_: arXiv:2408.09174. doi: [10.48550/arXiv.2408.09174](https://doi.org/10.48550/arXiv.2408.09174).

[^20]:
    Z. Zhang, X. Li, Y. Gao, and J.-G. Lou, "CRT-QA: A Dataset of Complex Reasoning Question Answering over Tabular
    Data," in _Proceedings of the 2023 Conference on Empirical Methods in Natural Language Processing_, H. Bouamor, J. Pino,
    and K. Bali, Eds., Singapore: Association for Computational Linguistics, Dec. 2023, pp. 2131–2153.
    doi: [10.18653/v1/2023.emnlp-main.132](https://doi.org/10.18653/v1/2023.emnlp-main.132).

[^21]:
    Y. Zhao, Y. Li, C. Li, and R. Zhang, "MultiHiertt: Numerical Reasoning over Multi Hierarchical Tabular and
    Textual Data," in _Proceedings of the 60th Annual Meeting of the Association for Computational Linguistics (Volume 1:
    Long Papers)_, S. Muresan, P. Nakov, and A. Villavicencio, Eds., Dublin, Ireland: Association for Computational
    Linguistics, May 2022, pp. 6588–6600.
    doi: [10.18653/v1/2022.acl-long.454](https://doi.org/10.18653/v1/2022.acl-long.454).

[^22]:
    D. Gomm and M. Hulsebos, "Metadata Matters in Dense Table Retrieval," in _ELLIS workshop on Representation
    Learning and Generative Models for Structured Data_, Feb. 2025.
