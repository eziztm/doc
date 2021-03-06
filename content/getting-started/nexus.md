Nexus
=====

We are running Nexus at NAV.


## Docker registry

You can push and pull from our registry at `repo.adeo.no:5443`, and visit it at [repo.adeo.no](https://repo.adeo.no/#browse/browse/components:docker).

You need to log into our Docker registry before pushing or pulling images:

```text
docker login repo.adeo.no:5443
docker push repo.adeo.no:5443/{application}:{releaseVersion}
```

Set up a build to tag with something unique but still useful, for instance date along with the git commit SHA.
We do not recommend using the tag `latest`. Read more about why [here](https://vsupalov.com/docker-latest-tag/).


## Maven repo

We use basic auth when uploading [NAIS manifest](/documentation/contracts/README.md#nais-manifest) to our Maven repo at [repo.adeo.no](https://repo.adeo.no/).

```
curl --user username:password --upload-file path/to/nais.yaml https://repo.adeo.no/repository/raw/nais/{appname}/{version}/nais.yaml
```
