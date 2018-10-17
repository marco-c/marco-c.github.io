---
title: Searchfox in Phabricator extension
tags: [review]
---

Being able to search code while reviewing can be really useful, but unfortunately it's not so straightforward. Many people resort to loading the patch under review in an IDE in order to be able to search code.

Being able to do it directly in the browser can make the workflow much smoother.

To support this use case, I've built an extension for Phabricator that integrates [Searchfox](https://searchfox.org/) code search functionality directly in Phabricator differentials. This way reviewers can benefit from **hovers**, **go-to-definition** and **find-references** without having to resort to the IDE or without having to manually navigate to the code on searchfox.org or dxr.mozilla.org. Moreover, compared to searchfox.org or dxr.mozilla.org, the extension highlights both the pre-patch view and the post-patch view, so reviewers can see how pre-existing variables/functions are being used after the patch.

To summarize, the features of the extension currently are:
1. Highlight keywords when you hover them, highlighting them both in the pre-patch and in the post-patch view;
2. When you press on a keyword, it offers options to search for the definition, callers, and so on (the results are opened on Searchfox in a new tab).

Here's a screenshot from the extension in action:
<figure>
  <img src="/assets/phabricator-mozsearch-addon.png" alt="Searchfox in Phabricator" />
  <figcaption><b>Figure 1</b>: Searchfox in Phabricator.</figcaption>
</figure>

I'm planning to add support for sticky highlighting and blame information (when hovering on the line number on the left side). 
Indeed, being able to look at the past history of a line is another sought after feature by reviewers.

You can find the extension on AMO, at [https://addons.mozilla.org/addon/searchfox-phabricator/](https://addons.mozilla.org/addon/searchfox-phabricator/).

The source code, admittedly not great as it was written as an experiment, lives at [https://github.com/marco-c/mozsearch-phabricator-addon](https://github.com/marco-c/mozsearch-phabricator-addon).

Should you find any issues, please file them on [https://github.com/marco-c/mozsearch-phabricator-addon/issues](https://github.com/marco-c/mozsearch-phabricator-addon/issues).
