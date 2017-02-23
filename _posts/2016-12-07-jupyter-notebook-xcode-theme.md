---
title: Jupyter Notebook Xcode theme
tags:
- Jupyter Notebook
- Xcode
- ML
crosspost_to_medium: true
---
So one Saturday I got particularly bored and thought I should configure my Jupyter Notebook a bit. <!--more--> What I wanted to achieve:

* Remove annoying Jupyter logo
* Use a better monospaced font
* Make syntax highlighting look like the Xcode default one

I'm not a huge fan of dark schemes and wanted to have some familiarity with Xcode syntax highlighting, so I forked [a neat looking solution](https://github.com/neilpanchal/iPython-Notebook-Theme) that already had header logo disabled, and then spent a couple of hours making sure the colours match to those of Xcode. 

Without further ado, this is what it looks like. I was pretty pleased with the result and have been using it ever since (e.g. for a couple of months now).

![image-center]({{ base_path }}/images/posts/jupyter-xcode-theme/jupyter-xcode-screenshot.png){: .align-center}

You can, of course, get this theme [on GitHub](https://github.com/navoshta/Jupyter-Notebook-Theme):

<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/navoshta" data-style="mega" data-count-href="/navoshta/followers" data-count-api="/users/navoshta#followers" data-count-aria-label="# followers on GitHub" aria-label="Follow @navoshta on GitHub">Follow @navoshta</a>
<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/navoshta/Jupyter-Notebook-Theme" data-icon="octicon-star" data-style="mega" data-count-href="/navoshta/Jupyter-Notebook-Theme/stargazers" data-count-api="/repos/navoshta/Jupyter-Notebook-Theme#stargazers_count" data-count-aria-label="# stargazers on GitHub" aria-label="Star navoshta/Jupyter-Notebook-Theme on GitHub">Star</a>
<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/navoshta/Jupyter-Notebook-Theme/fork" data-icon="octicon-repo-forked" data-style="mega" data-count-href="/navoshta/Jupyter-Notebook-Theme/network" data-count-api="/repos/navoshta/Jupyter-Notebook-Theme#forks_count" data-count-aria-label="# forks on GitHub" aria-label="Fork navoshta/Jupyter-Notebook-Theme on GitHub">Fork</a>
<!-- Place this tag where you want the button to render. -->
<a class="github-button" href="https://github.com/navoshta/Jupyter-Notebook-Theme/archive/master.zip" data-icon="octicon-cloud-download" data-style="mega" aria-label="Download navoshta/Jupyter-Notebook-Theme on GitHub">Download</a>

<!-- Place this tag in your head or just before your close body tag. -->
<script async defer src="https://buttons.github.io/buttons.js"></script>

 You can find detailed installation instructions following [this link](https://github.com/nsonnad/base16-ipython-notebook). Although for me it was as easy as placing the `custom.css` file in the `~/.jupyter/custom` folder on both my macOS and Ubuntu machines.

Final note: if you don't have Apple's `Menlo` font installed you may want to check out its open source alternative [`Meslo`](https://github.com/andreberg/Meslo-Font).