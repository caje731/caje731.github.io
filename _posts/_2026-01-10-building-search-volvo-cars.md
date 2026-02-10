---
title: "Building Search at Volvo Cars"
categories:
  - Blog
tags:
- management
- leadership
- technology
---

What does it take to build a search engine for a global car company?

With a hundred-year legacy and a commitment to innovation, Volvo Cars has always been at the forefront of automotive technology. In recent years, the company finally recognized the importance of giving customers a seamless and efficient search experience across our digital platforms, coming on the back of repeated customer feedback and falling CSAT scores. This realization led to the establishment of a Search team, tasked with developing and maintaining our search capabilities, and I was privileged to be the Engineering Manager delivering this critical function. In this blog post, I will share insights into some key technological decisions I helped the team with, the trade-offs they came with, and the impact they had on our search capabilities.

## Choosing the Right Search Technology

One of the first decisions we faced was selecting the right search technology. Elasticsearch was already a partner to Volvo Cars for observability, providing a level of familiarity and strong community support. In hindsight, this was a great choice at the time, as it allowed us to leverage existing expertise and resources. However, we also had to consider the trade-offs, such as the learning curve for team members who were new to Elasticsearch and the need for additional infrastructure to support it. I'm talking not just the search engine itself, but also the data pipelines and indexing processes required to keep our search data up-to-date. In hindsight, I think I did the best I could with the people and resources I had at the time, and we were able to go to production within a matter of 2-3 months, which was a significant achievement for the team.

## The Corpus Challenge

With pages in 120 market-locales and a total of >2 million documents, precision and recall were critical metrics for our search engine. We had to ensure that our search results were relevant and accurate, which required careful tuning of our search algorithms and continuous monitoring of performance. This was a significant challenge, as we had to balance the need for precision with the need for recall, ensuring that we were not missing relevant results while also avoiding irrelevant ones. We also had to consider the impact of language and cultural differences across our global markets, which added another layer of complexity to our search capabilities.
