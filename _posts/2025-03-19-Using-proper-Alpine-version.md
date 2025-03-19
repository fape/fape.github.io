---
layout: post
title: Megfelelő Alpine linux verzió használata csomag telepítéshez CI/CD vagy Docker esetén
---

# Probléma
Hogyan lehet az aktuális [Alpine linux](https://alpinelinux.org/) verzió számot megkapni és felhasználni saját apk (proxy) repo (pl [artifactory](https://jfrog.com/artifactory/)) esetén. Pl ha az image version csak `22-alpine`. 

# Ötlet #1
Mit használnak Alpine-ék?
[Ezt:](https://gitlab.alpinelinux.org/alpine/alpine-conf/-/blob/master/setup-apkrepos.in?ref_type=heads#L127)
```bash
get_alpine_release() {
	# use the main version already configured, or get the version from /etc/alpine-release
	local version="$(grep -Eom1 '[^/]+/main/?$' "${ROOT}"etc/apk/repositories 2>/dev/null | grep -Eo '^[^/]+' \
		|| cat "${ROOT}"etc/alpine-release 2>/dev/null)"
	case "$version" in
		*_git*|*_alpha*) release="edge";;
		[0-9]*.[0-9]*.[0-9]*)
			# release in x.y.z format, cut last digit
			release=v${version%.[0-9]*};;
		v[0-9]*.[0-9]*)
            # release in vx.y format, keep as is
			release="${version}";;
		*)	# fallback to edge
			release="edge";;
	esac
}
```

# Megoldás 1
Ebből saját egyszerűsített megoldás CI/CD-hez alpine-s node-s image-hez

```yaml
build:
  image: ${BASE_DOCKER_IMAGE_REPOSITORY}/docker-remote/node:22-alpine
  stage: build
  #when: manual
  before_script:
    - repo_version="$(grep -Eom1 '[^/]+/main/?$' /etc/apk/repositories 2>/dev/null | grep -Eo '^[^/]+')"
    - echo "https://sajat_artifactory/${repo_version}/main" > /etc/apk/repositories
    - echo "https://sajat_artifactory/${repo_version}/community" >> /etc/apk/repositories
    - apk update --no-cache
    - apk add --no-cache zip rsync

```
# Megoldás 2
Vagy másik megoldás pl Docker fájlhoz
```bash
echo -e "https://sajat_artifactoryv$(cut -d . -f 1,2 < /etc/alpine-release)/community\n
	https://sajat_artifactory/v$(cut -d . -f 1,2 < /etc/alpine-release)/main" > /etc/apk/repositories
```
