# BPM on Windows
- Image? Can we bind mount the host file system? What does that mean?
  - Bind mounting is ideal and solves the licenscing/packaging problems
- Isolation primitives are not the same
  - Namespaces do not exist
  - Seccomp, apparmor, do not exist
- WINC adheres to runtime-space (need to implement the root process)
- Monit file is identical to a subset of the `bpm.yml`
- Mounts will differ between OS
  - Mounts is not OS configurtion, therefore we will need to branch on windows executable to not mount all linux mounts.
  - Appears that BOSH mounts are the same, `C:\\var\\vcap\data`...
- Do we support 2016 only? 2012 doesn't support WINC only Ironframe
  - If we choose to use garden-windows this problem could be solved for us (Ironframe is fake containers)

# Transition
- BOSH windows respects both `monit` or `config/bpm.yml`
- Implement a v1 BPM on Windows
- BOSH windows service wrapper shells out to BPM for it's lifecycle

# Personal Opinions
- If we need to have a base image to use WINC, I think it may be too heavy weight to use.
  - Instead we may consider executing windows processes without a container for the time being.
