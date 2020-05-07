# Machine Learning Study

Notes, code and useful assets about machine learning.

Tools and tips:

* [Typora](https://typora.io/) for markdown editing.

* [Deepo](https://hub.docker.com/r/ufoym/deepo) (more helpful to check the Docker Hub page instead of its home page) for Jupyter and other Python ML environment. All the uploaded Jupyter notebooks are made and tested with [this](https://hub.docker.com/layers/ufoym/deepo/all-jupyter-py36-cpu/images/sha256-372e2014ed6a6dc6c96f74880339e3cf75116aa876a55e3a29c971d6c951adee?context=explore) Deepo setup.

* [Working with Docker Desktop (Windows 10) and WSL](https://nickjanetakis.com/blog/setting-up-docker-for-windows-and-wsl-to-work-flawlessly#ensure-volume-mounts-work).

* [Tips for running Docker, Deepo and Jupyter](https://zhuanlan.zhihu.com/p/64493662).

* RL algorithms baselines: [OpenAI Baselines](https://github.com/openai/baselines) (with a Tensorflow 2.0 branch) and [Stable Baselines](https://stable-baselines.readthedocs.io/) (used in Kaggle courses, well documented and maintained, highly recommended).

Learning material:

* David Silver/UCL reinforcement learning lectures ([slides](https://www.davidsilver.uk/teaching/), [video](https://www.bilibili.com/video/BV1kb411i7KG?from=search&seid=10277587044180293066)), excellent resource for learning RL. Downloaded slies are in folder "David_Silver_RL_lectures". My notes on the lectures is in David_Silver_UCL_RL_lectures_notes.md.

* [Deep Reinforcement Learning Through Policy Optimization (video)](https://channel9.msdn.com/events/Neural-Information-Processing-Systems-Conference/Neural-Information-Processing-Systems-Conference-NIPS-2016/Deep-Reinforcement-Learning-Through-Policy-Optimization?term=policy%20gradient&lang-en=true), introduction to various advanced policy gradient and actor/critic methods, downloaded slide is in deep-reinforcement-learning-through-policy-optimization.pdf.

* [Going Deeper Into Reinforcement Learning: Fundamentals of Policy Gradients](https://danieltakeshi.github.io/2017/03/28/going-deeper-into-reinforcement-learning-fundamentals-of-policy-gradients/), very detailed mathematical explanation on log-derivative in score function and the reason why introducing baseline (advantage function) can reduce variance while keeping bias unchanged. Downloaded version is in Fundamentals-of-Policy-Gradients.pdf.

* [Policy Gradient Algorithms](https://lilianweng.github.io/lil-log/2018/04/08/policy-gradient-algorithms.html), a deep look into almost all mainstream policy gradient algorithms proposed in recent years. Downloaded version is in Policy-Gradient-Algorithms.pdf.

* [AlphaZero Gomoku](https://github.com/junxiaosong/AlphaZero_Gomoku), very good resource for understanding the principle of AlphaZero. My notes on this project is in AlphaZero_Gomoku_notes.md.

* [Cross-entropy error instead of MSE for classification problem](https://jamesmccaffrey.wordpress.com/2013/11/05/why-you-should-use-cross-entropy-error-instead-of-classification-error-or-mean-squared-error-for-neural-network-classifier-training/), well explained.

* Liu Jianping's machine learning [blog](https://www.cnblogs.com/pinard/), detailed notes for various topics on machine learning.
