# URL Analysis 101

> How phishing URLs are manipulated to look trustworthy, from cybersquatting to short URLs, and how to detect them at scale with machine learning.

- Source: https://research.intezer.com/docs/url-analysis/
- Published: 2022-03-03
- Attribution: Intezer's Research Team (Intezer Research, https://research.intezer.com/)


A malicious link is often all an attacker needs. Phishing URLs are built to
survive a quick glance: a trusted brand buried in a subdomain, a single missing
letter, a top-level domain nobody registered, or a redirect hidden behind a
shortener. Reading a URL correctly is a skill, and at scale it has to be
automated.

This section covers both halves of the problem — spotting manipulation by eye,
and detecting it in volume.

- [A Beginner's Guide to Phishing URLs](part1/) - The anatomy of a URL and why
  it should be read right to left, followed by the manipulation techniques that
  exploit it: subdomain, TLD and typo cybersquatting, suggestive words, soft
  hyphens, hex encoding, short URLs, and HTML concealment.
- [Automating Phishing Investigations with Machine Learning](part2/) -
  List-based detection versus machine learning, the trade-offs of each, and a
  worked example turning URLs into feature vectors and separating phishing from
  trusted with an SVM.

## Pages in this section

- [A Beginner's Guide to Phishing URLs](https://research.intezer.com/docs/url-analysis/part1/): The anatomy of a URL and the tricks phishing links use: subdomain, TLD and typo cybersquatting, special characters, short URLs and HTML concealment.
- [Automating Phishing Investigations with Machine Learning](https://research.intezer.com/docs/url-analysis/part2/): List-based versus machine learning detection of phishing URLs, the trade-offs of each, and extracting URL features to classify phishing links with an SVM.

