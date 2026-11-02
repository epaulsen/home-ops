When adding or changing a container image in a Kubernetes manifest, verify that
the exact image tag exists in its upstream registry before using it. Do not
assume an application version alone is a valid image tag; image tags may include
additional runtime or distribution suffixes.
