# Announcing Kthena 0.4.0

Thanks to the incredible dedication and collective efforts of our contributors over the past two months, Kthena’s stability has reached new heights. We want to express our deepest gratitude to everyone who contributed to this milestone. Today, we are thrilled to announce the official release of **Kthena 0.4.0**—our most robust and feature-rich version yet!

Beyond rock-solid stability, Kthena 0.4.0 introduces a wave of exciting new features designed to streamline your LLM workloads and empower your AI infrastructure.

## Unprecedented Observability

To minimize kube apiserver load, Kthena's `ModelServing` utilizes a local store to cache the status of `ServingGroup`s and `Role`s. While highly efficient, we realized this temporarily limited observability during debugging.

In v0.4.0, we've broken the black box. We now [expose role status directly via Kubernetes Events](https://github.com/volcano-sh/kthena/pull/676), dramatically enhancing the observability of `ModelServing`. Looking ahead, we plan to directly embed this crucial Role information into the `ModelServing` Status, giving you complete, transparent control over your deployments.

## A Faster, Smarter Router

### Deterministic & Efficient Model Selection

Previously, mapping multiple `ModelRoute` resources to a single model could trigger route conflicts—leading to ambiguous rule matching and inconsistent target selection. Because Kubernetes' built-in CRD validation cannot enforce global cross-object uniqueness, we tackled this gracefully at the routing layer.

Kthena 0.4.0 introduces a robust [conflict-resolution mechanism](https://github.com/volcano-sh/kthena/pull/779). When duplicate `ModelRoute`s exist, the router deterministically prioritizes the oldest (typically prebuilt) route, treating newer duplicates as lower priority. The result? Predictable, rock-solid routing every time.

### Configurable Prefix-Caching

One size doesn't fit all. That's why we replaced hardcoded prefix-cache parameters with a fully [configurable prefix-matching system](https://github.com/volcano-sh/kthena/pull/844). You now have granular control over block sizes for hash processing, maximum block limits, cache capacity, and top-K results. By fine-tuning the prefix-cache, you can strictly tailor Kthena's performance to match your specific models and business workloads perfectly.

### Comprehensive Access Logging

Troubleshooting routing issues is now easier than ever. The Router now generates a highly detailed access log, capturing rich [routing metadata](https://github.com/volcano-sh/kthena/pull/621) for every single request. Say goodbye to guesswork when debugging complex traffic resolutions.

## Granular, Resource-Efficient Rolling Updates

Historically, Kthena executed rolling updates at the entire `ServingGroup` level. For massive LLM applications, completely rebuilding a `ServingGroup` is an incredibly resource-intensive and time-consuming process.

To solve this, we are introducing [role-based rolling updates](https://github.com/volcano-sh/kthena/pull/802). You no longer need to update an entire `ServingGroup` when only a specific `Role` requires changes (which is also why we introduced `RoleCreate` in the recovery policy). Starting from 0.4.0, you can dynamically adjust your `rolloutStrategy`—drastically cutting down resource consumption, speeding up deployment times, and maximizing the high availability of your LLM applications.

## A Thriving, Connected Ecosystem

We are deeply committed to building Kthena alongside the wider open-source community. In 0.4.0, we have supercharged our model downloader with [ModelScope protocol support](https://github.com/volcano-sh/kthena/pull/861) to provide a seamless, high-speed alternative to Hugging Face for developers facing network constraints in China.

Additionally, `ModelServing` is now thoroughly verified to support leading inference engines like **vLLM** and **sglang**. As Kthena continues to weave deeper into the Cloud Native AI ecosystem, we warmly invite anyone interested to join us in building this vibrant future together!
