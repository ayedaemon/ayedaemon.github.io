---
title: "Hands-on Intro to SBOM"
date: 2022-05-27T22:56:22+05:30
draft: false
# showtoc: false
tags: [sbom, docker, security]
# series:
# description:
# math: true
# ShowBreadCrumbs: false
# searchHidden: true
# hideMeta: true
---

The concept of a Bill Of Materials (BOM) is well-established in traditional manufacturing as part of supply chain management. A manufacturer uses a BOM to track the parts it uses to create a product. If defects are later found in a specific part, the BOM makes it easy to locate affected products. In software industry, this concept is fairly new and is used to keep track of all the ingredients of the software.

### What is SBOM ??

A software bill of materials (SBOM) is a formal record of the components used to develop software and its software supply chain relationships, according to the National Telecommunications and Information Administration (NTIA). An SBOM covers both open source (OSS) and proprietary software, creating transparency into potential vulnerabilities and elements within the software. SBOMs can be used for vulnerability management and product integrity.

An SBOM is useful both to the builder (manufacturer) and the buyer (customer) of a software product. Builders often leverage available open source and third-party software components to create a product; an SBOM allows the builder to make sure those components are up to date and to respond quickly to new vulnerabilities. Buyers can use an SBOM to perform vulnerability or license analysis, both of which can be used to evaluate risk in a product.


### Why SBOM ??

There could be multiple usages of SBOM, like

- easy End-Of-Life management for dependencies and product itself.
- License obligations and policy compliance.
- For developers, it can help to unbloat the software by identifying the BOM and clean up unused things or can use it for quality assurance.
- Identify and eliminate vulnerabilities from early stages (more shift left)

There are many artifacts that can provide SBOM information and this information can be correlated and used together to provide better security insights. These artifacts could be the source code, executables,  published softwares, or in devops world, **containers**!!

Containers are easy way to package and deliver software; Container is like an encapsulated artifact. Here we can get SBOM for **Application dependencies, Secret code, OS packages, Licenses, File data, Configuration files, Container meta-data, etc**. When it comes to security, it’s important to know every part of the system. SBOM gives you a clear list of components that help in monitoring every part for vulnerabilities.


### Existing SBOM formats
A new SBOM can be created and published in various formats including HTML, CSV, PDF, Markdown, and plain text. SBOM formats are still in development and new formats might arise in future that can address specific problems in a better way. Currently used formats are -  Software Package Data Exchange (SPDX), Software Identification (SWID) Tags, and Cyclone DX.

1. [**SPDX**](https://spdx.dev/)

Also known as ISO/IEC 5962:2021, SPDX is spearheaded by The Linux Foundation. It is an open standard for describing SBOM information related to provenance, licensing, and security.

2. [**SWID Tags**](https://csrc.nist.gov/projects/Software-Identification-SWID)

This format identifies and reports software components under four categories across the development lifecycle:

- Corpus Tags: Identifies and describes components in a pre-installation stage.
- Primary Tags: Identifies and describes components in a post-installation stage.
- Patch Tags: Identifies and describes the patch.
- Supplement Tags: Allows only the tag creator to modify corpus, primary, and patch tags.

3. [**Cyclone DX**](https://cyclonedx.org/)

Managed by Cyclone DX’s core working group, it is designed for application security contexts. Cyclone DX is considered a lightweight standard with features of both SPDX and SWID. It includes four data fields:

- **BOM Metadata**: Description of the supplier, manufacturer, component, and compilation tools.
- **Components**: Complete information of a proprietary and open-source components along with licensing requirements.
- **Services**: A list of external APIs that the software may invoke.
- **Dependencies**: All forms of relationship within the supply chain.



### Don't talk, show!!

For the demo, I've created a basic flask application that says hello and have containerized it into 3 different base images - ubuntu, alpine and distroless.

```shell{linenos=false}
CONTAINER ID   IMAGE             COMMAND                  CREATED          STATUS          PORTS     NAMES
783618b1c6df   sbom_distroless   "/usr/bin/python3.9 …"   9 seconds ago    Up 7 seconds              sbom_distroless_demo
3ed64aef4767   sbom_alpine       "python app.py"          16 seconds ago   Up 14 seconds             sbom_alpine_demo
fe18c421777a   sbom_ubuntu       "python3 app.py"         19 seconds ago   Up 17 seconds             sbom_ubuntu_demo
```

We can check the size of the container image using 	`docker images` command.

```txt{linenos=false}
sbom_distroless             latest       6ef7ccd61f84   38 minutes ago      166MB
sbom_alpine                 latest       e7e71b412cf5   About an hour ago   161MB
sbom_ubuntu                 latest       9e2166292230   About an hour ago   573MB
```

If you want to get more details about the size of each layer then you can use `docker history <image>` command. More information about the running container (process) can be obtained using `docker inspect <container>`.

All these commands are good, but they do not provide any information about the application and its dependencies. Docker has recently announced its experimental feature - `docker sbom`, that allows us to generate the SBOM of a container image. Today, it does this by scanning the layers of the image using the [**Syft**](https://github.com/anchore/syft) project but in future it may read the SBOM from the image itself or elsewhere.

Let's generate a SBOM for our containers by directly using the syft project.

```txt{linenos=false}
syft sbom_distroless
 ✔ Loaded image
 ✔ Parsed image
 ✔ Cataloged packages      [69 packages]
NAME                  VERSION                       TYPE
Flask                 2.1.2                         python
Jinja2                3.1.2                         python
MarkupSafe            2.1.1                         python
Werkzeug              2.1.2                         python
base-files            11.1+deb11u3                  deb
boto3                 1.23.9                        python
botocore              1.26.9                        python
certifi               2022.5.18.1                   python
charset-normalizer    2.0.12                        python
click                 8.1.3                         python
dash                  0.5.11+git20200708+dd9ef66-5  deb
idna                  3.3                           python
importlib-metadata    4.11.4                        python
itsdangerous          2.1.2                         python
jmespath              1.0.0                         python
libbz2-1.0            1.0.8-4                       deb
libc-bin              2.31-13+deb11u3               deb
libc6                 2.31-13+deb11u3               deb
libcom-err2           1.46.2-2                      deb
libcrypt1             1:4.4.18-4                    deb
libdb5.3              5.3.28+dfsg1-0.8              deb
libexpat1             2.2.10-2+deb11u3              deb
libffi7               3.3-6                         deb
libgcc-s1             10.2.1-6                      deb
libgomp1              10.2.1-6                      deb
libgssapi-krb5-2      1.18.3-6+deb11u1              deb
libk5crypto3          1.18.3-6+deb11u1              deb
libkeyutils1          1.6.1-2                       deb
libkrb5-3             1.18.3-6+deb11u1              deb
libkrb5support0       1.18.3-6+deb11u1              deb
liblzma5              5.2.5-2.1~deb11u1             deb
libmpdec3             2.5.1-1                       deb
libncursesw6          6.2+20201114-2                deb
libnsl2               1.3.0-2                       deb
libpython3.9-minimal  3.9.2-1                       deb
libreadline8          8.1-1                         deb
libsqlite3-0          3.34.1-3                      deb
libssl1.1             1.1.1n-0+deb11u2              deb
libstdc++6            10.2.1-6                      deb
libtinfo6             6.2+20201114-2                deb
libtirpc3             1.3.1-1                       deb
libuuid1              2.36.1-8+deb11u1              deb
netbase               6.3                           deb
openssl               1.1.1n-0+deb11u2              deb
pip                   22.0.4                        python
pip                   22.1.1                        python
python-dateutil       2.8.2                         python
python3-distutils     3.9.2-1                       deb
requests              2.27.1                        python
s3transfer            0.5.2                         python
setuptools            58.1.0                        python
six                   1.16.0                        python
tzdata                2021a-1+deb11u3               deb
urllib3               1.26.9                        python
wheel                 0.37.1                        python
zipp                  3.8.0                         python
zlib1g                1:1.2.11.dfsg-2+deb11u1       deb
```

By default, syft parses and analyses the final layer of the container and displays the tabular result on the standard output (stdout). This is good if we just want to see the SBOM ourselves and not want to share it with other tools or people. To save the output to a file you can use `--file` option and you can also specify another formats that are widely used by community with `-o` or `--output` flag. Below bash script will create `cyclonedx-json` , `github-json`,  `spdx-json`and `syft-json`  format SBOMs and also store them in their respective files.

```bash
mkdir -p generated_sboms;
for i in sbom_{ubuntu,distroless,alpine};
do
mkdir -p generated_sboms/$i
echo $i;
syft $i \
	-o syft-json=generated_sboms/$i/syft.json \
	-o spdx-json=generated_sboms/$i/spdx.json \
	-o github-json=generated_sboms/$i/github.json \
	-o cyclonedx-json=generated_sboms/$i/cyclonedx.json
done
```
Output of the above script provides us with package count for each image and it is clear that the ubuntu has most of them as it is a full fledged distro with a lot of system files, manpages, etc... and distroless images have the least one. The idea of distroless is somewhat over-hyped in the world of containers and sometimes it can be related with
security ideas of minimum attack surface. [Here is a RedHat article](https://www.redhat.com/en/blog/why-distroless-containers-arent-security-solution-you-think-they-are) that try to give a clear understanding of the benefits of distroless containers and myths around it.

```txt{linenos=false}
sbom_ubuntu
 ✔ Loaded image
 ✔ Parsed image
 ✔ Cataloged packages      [265 packages]
sbom_distroless
 ✔ Loaded image
 ✔ Parsed image
 ✔ Cataloged packages      [69 packages]
sbom_alpine
 ✔ Loaded image
 ✔ Parsed image
 ✔ Cataloged packages      [71 packages]
```
And it'll create a directory with organised json files

```txt{linenos=false}
# tree generated_sboms/

generated_sboms/
├── sbom_alpine
│   ├── cyclonedx.json
│   ├── github.json
│   ├── spdx.json
│   └── syft.json
├── sbom_distroless
│   ├── cyclonedx.json
│   ├── github.json
│   ├── spdx.json
│   └── syft.json
└── sbom_ubuntu
    ├── cyclonedx.json
    ├── github.json
    ├── spdx.json
    └── syft.json
```

Now we have our sbom files and we can share these files to other people who need it. It can be our customers, external auditors, Incident response team, etc etc... Also we can use these files with another tool that can check these images for vulnerabilities. One such tool is [**grype**](https://github.com/anchore/grype) - A vulnerability scanner for container images and filesystems that works exceptionally with Syft. Below script will generate grype results for all the 3 images using their respective `spdx.json` files.

```bash
mkdir -p grype_results;
for i in sbom_{ubuntu,distroless,alpine};
do
echo $i;
mkdir -p grype_results/$i
grype sbom:./generated_sboms/sbom_ubuntu/spdx.json \
	-o json \
	--file grype_results/$i/all.json
done
```

Like all static analysers, this tool might generate tons of false positives. Apart from this, grype tool provides tons of configuration features that can come in handy for automations and several other usecases. A lot of other commercial and open-source tools are arising that can leverage SBOMs and can help to solve problems around licencing and policy compliene, security audits, quality assurance, etc.

### SBOM misconception

There are few misconeptions or myths about SBOMs like it can :-

1. be a roadmap to the attacker ?
2. require source code disclosure ?
3. expose my intellectual properties ? .. etc

[Here is a NTIA publication](https://www.ntia.gov/files/ntia/publications/sbom_myths_vs_facts_nov2021.pdf) that covers explaination of some such myths V/S facts.
