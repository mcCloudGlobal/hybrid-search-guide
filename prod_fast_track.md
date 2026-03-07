# ADK fast track to production

## 1. Enhance hybrid search agent with agent-starter-pack

Navigate to *hybrid search agent* folder: `cd ~/hybrid-search-guide/hybrid_search` and install [agent-starter-pack](https://github.com/GoogleCloudPlatform/agent-starter-pack) with option enhance:
```bash
cd ~/hybrid-search-guide
uvx agent-starter-pack enhance
```

Choose target project and deployment environment (I recommend Agent Engine to start with).  
Review deployment options, including Terraform IaC for full IaC deployment.  

Build and start adk web playground:

```bash
make install && make playground
```

When ready to proceed deploy using:

```bash
make deploy
```

Full guide is [here](https://googlecloudplatform.github.io/agent-starter-pack/guide/development-guide.html).

## 2. Read more to understand agent-starter-pack value

Go to [Why an Agent Starter Pack?](https://googlecloudplatform.github.io/agent-starter-pack/guide/why_starter_pack.html) to learn more. Read [docs](https://googlecloudplatform.github.io/agent-starter-pack/guide/getting-started.html) and watch [videos](https://googlecloudplatform.github.io/agent-starter-pack/guide/video-tutorials.html) to use full potential of thi framework.
