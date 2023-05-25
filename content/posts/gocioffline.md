+++
title = "Building a Go CI pipeline in an offline environment"
author = "Runit Misra"
date = 2023-05-25
+++

In today's interconnected world, the majority of software development and deployment processes heavily rely on internet connectivity. However, certain organizations dealing with highly sensitive data cannot afford to expose their infrastructure to the internet or must tightly control internet access. This is where Offline environments, also known as Airgapped environments, come into play. Offline environments refer to infrastructure with limited or no connectivity to the open internet. While they offer enhanced security and independence from internet availability, they pose unique challenges for building CI (Continuous Integration) pipelines, especially when it comes to managing third-party dependencies. In this blog post, I will share my experiences and insights on overcoming this challenge, exploring various solutions tailored to specific situations.

The task at hand is straightforward: construct a CI pipeline that retrieves the latest code from the application's repository, executes unit tests, and creates a Docker image for the application. However, building such a pipeline becomes challenging in an offline environment where accessing dependencies from external sources like GitHub or Go's package registry is not feasible. These dependencies are required both during test runs and application builds. In the following sections, we will explore effective solutions to tackle this issue and ensure a successful CI pipeline in an offline environment.

## Use a private registry
This is by far the best solution and the one that almost anyone faced with this situation will opt for. Artifact repositories make life very simple for engineers looking for fine-grained control over their software supply chain, simplifying the management of organization-generated artifacts and enabling controlled retrieval of artifacts from the internet, ensuring they meet necessary security requirements.

There are some excellent choices out there like JFrog's Artifactory or Sonartype's Nexus repository. You can use them as proxy to pull any required package from a list of verified sources which would be controlled by you. You can also upload packages manually to these repositories after through security scans and policy checks to make sure each package meets the security standards you have set. Once set up, private registries can be useful for multiple teams across the organization to store artifacts, track their versions and securely access them in an offline environment.

More information available here:

**Artifactory:** https://jfrog.com/help/r/jfrog-artifactory-documentation/go-registry

**Nexus:** https://help.sonatype.com/repomanager3/nexus-repository-administration/formats/go-repositories

## Use go vendoring
This might seem like a hack because it probably is. Vendoring, if you aren't familiar, enables you to commit the dependencies next to the code. Vendoring is used to maintain the deterministic nature of go builds in an event a package your application depends on might get deleted/moved/broken later down the line. 

However, this works out great for us because when vendoring is enabled, go uses the packages it downloaded in the `vendor` directory to satisfy the application dependencies. This download can be done on a machine with internet access like the developer's PC using the `go mod vendor` command. Once the `vendor` directory is created, it can be committed alongside the code. The CI pipeline can then pull the dependencies directly from the `vendor` directory because it is part of the code repository and hence there is no need for connecting to the internet.

Now that vendoring is set up, use `-mod=vendor` argument in all the build commands in the CI pipeline. For example:
- To test use `go test -mod=vendor ./...`
- To build use `go build -mod=vendor cmd/main.go`

## Use a base image
Here, by "Base image" I mean a docker image you prepare beforehand that has the necessary dependencies already downloaded and baked into it. Here the `go mod download` command would come in handy. Just like vendoring, the dependencies can be downloaded from a machine with internet access but instead of maintaining the packages in the repository along side the code, the packages are downloaded in a docker image which is then used as the base for the final application docker image.

Create the base image with a dockerfile looking something like this:
```Dockerfile
FROM golang:alpine
RUN go mod download -x
```
You can tag this something like `custom-base:latest`

Then the application Dockerfile will look something like this:
```Dockerfile
FROM custom-base:latest AS builder
COPY . /build
RUN go build -o /build/myapp

FROM alpine:latest
COPY --from=builder /build/myapp /app
CMD ["./app/myapp"]
```

To be honest, this is not the most optimal solution. The biggest overhead this will create for you is to re-build this image in case any dependency gets updated, added or removed. You could always automate this but it has to be done on a machine with internet access and you'd have to spend time writing logic when better alternatives are available.

## Conclusion
Not being able to rely on the convenience of having internet connectivity can be a little frustrating in day-to-day operations. But convenience sometimes comes with a cost - in our case it is fine grained control over access to internet to maintain security and privacy standards. However, these methods can hopefully help you in overcoming the challenge of CI in offline environments. 