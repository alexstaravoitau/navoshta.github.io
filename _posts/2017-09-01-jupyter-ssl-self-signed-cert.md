---
title: Self-signed SSL certificate in Jupyter
tags:
- Python
- ML
crosspost_to_medium: false
link: https://kernels.io/ssl-self-signed-cert/
---

In order to use Jupyter Notebook on iPad, one needs to correctly configure SSL certificates. Since issuing a proper certificate from a trusted authority could be challenging in some cases, a self-signed certificate should suffice, provided it was signed by a CA that is trusted by device. Follow these steps to get it working on your iPad!