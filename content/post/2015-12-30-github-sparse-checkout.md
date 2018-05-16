+++
author = "Viton Vitanis"
date = "2015-12-30"
linktitle = "Git - Sparse Checkout"
title = "Git - Sparse Checkout"
description = "Checkout only part of a git repository"
tags = [
  "git",
  "github"
  ]
categories = [
  "development"
]
weight = 10
+++
The other day I wanted to checkout only part of a github repository, to be more specific, the Spark example subfolder of the [kubernetes github repository](https://github.com/kubernetes/kubernetes): [tree/master/examples/spark](https://github.com/kubernetes/kubernetes/tree/master/examples/spark). I followed the instructions in this [Stackoverflow answer](http://stackoverflow.com/questions/600079/is-there-any-way-to-clone-a-git-repositorys-sub-directory-only):
Create the folder where the repository code will be stored:
{{< highlight bash >}}
git init
git remote add -f origin git://github.com/kubernetes/kubernetes.git
{{< /highlight >}}
Set _core.sparseCheckout_ to _true_:
{{< highlight bash >}}
git config core.sparseCheckout true
{{< /highlight >}}
and define which files should actually be checkout out:
{{< highlight bash >}}
echo "examples/spark" > .git/info/sparse-checkout
{{< /highlight >}}
Finally update the (empty) repo with the state from the remote:
{{< highlight bash >}}
git pull origin master
{{< /highlight >}}

Done!

