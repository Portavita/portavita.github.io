---
layout: post
title:  "OWASP Dependency Checks and Management of Software of Unknown Provenance"
date:   2019-11-19 10:01:00 +0200
categories: owasp soup
author: Wouter van Teijlingen
---

#### GLOSSARY

OWASP Open Web Application Security Project

SOUP Software of Unknown Provenance

#### Introduction

In this article I'll try to explain what tools we use to keep our software safe and meet regulatory requirements.

First things first: OWASP dependency check [1] is a tool to check if any security issues exists for the dependencies in your software projects. SOUP [2] is software that has it origins either inside or outside your organization but is - potentially - developed in such a way that it doesn’t meet regulatory requirements.

Okay, I’ve provided some working definitions of the OWASP dependency checks and SOUP management. The couple exists of two distinct subjects but in this article I will dive into some details about how we relate one to the other.

#### OWASP Dependency Checks

We use the dependency check [3] plugin for Jenkins and each night we automatically collect dependency check reports. The plugin checks the report for potential security issues and we set an acceptance threshold for security warnings (i.e., our boundary is set at 0). This allows our teams to actively monitor potential security issues: red means problems and requires investigation whereas green means all is OK. One of the disadvantages is that you have to deal with false positives. In many of our projects we have to deal with suppressions to get things back in order. As far as we’re concerned a small price to pay for the upside: active engagement in tasks to enhance safety of our products.

#### Management of SOUP

I think that management of software dependencies is important. It’s shocking to see that some projects use hundreds of dependencies for trivial tasks such as entering some text in a UI. Somewhat amusing to me is that software engineers that I know personally scrutinize their own and colleagues code but they have a don’t care attitude towards externals dependencies. I think that’s odd: often the volume of code in your dependencies is higher than the code you actually write yourself. As a side note I want to mention Ken Thompson. He wrote about trusting trust [4] a couple of decades ago and I think it’s an important paper to read as developer: to make sure you understand the consequences of such an attitude towards software written by others. Fact of developer life is that we cannot deliver much without usage of dependencies.

Anyway, this - and the fact that we are required to do so - is why we at Portavita have decided to take on SOUP management. There are many options that you can pick to implement SOUP. The thing is: setting this up is very much organization specific. You have to consider development workflows, build infrastructure, organization structure and developers.

SOUP management is about scrutinizing your dependencies. There are three classes: A, B, and C. Class A is the easiest to administer and C is the hardest [5]. What we need is an automated way to collect dependencies of our software projects and a tool to record details of the dependencies that we use. In the following section I’ll explain how we use the OWASP dependency checks to feed our SOUP management tool.

#### OWASP and SOUP Integration

I already mentioned the OWASP dependency check reports. The reports provide some details that I require for SOUP management: name and version of software dependency and if available license information.

I’ve created a jenkins dashboard with builds for each of our software projects. These builds send the OWASP dependency check reports to a service running on our kubernetes platform. It’s very simple: it’s only task is to consume those report and make sure it’s associated with the right project. It’s response is true or false. True means that all dependencies meet our SOUP management criteria and false if not. This makes it pretty simple for developers to see whether their projects meet SOUP requirements.

For developers it’s possible to administer the dependencies in a simple front-end built with polymer webcomponents. For those of you interested: also runs on our kubernetes platform. It implements a very straight-forward authentication scheme and each developer has a set of credentials. We need this otherwise we don’t have the required traceability.

We also keep a list of approved licenses that are automatically cross checked with the licenses of our dependencies. If a license of a dependency is unknown or invalid it is impossible to approve the dependency.

The downside of all this is that it still requires manual effort: a developer needs to check the dependencies and decide whether the risk introduced or not is acceptable. As it stands right now it’s part of our job to make such decisions.

#### Future Work

All is well and everybody happy… I wish. There are some problems that require further software and product engineering to make the SOUP management system more resilient.

The OWASP dependency check reports are problematic. The structure of dependency names, versions and licenses vary a lot. This makes it (very) hard to parse such data. I assume a better approach is to use reports generated by build tool specific utilities to generate dependency overviews. At minimum it makes parsing a lot easier.

The account management model is probably too basic. Also: it’s separate from our other account systems meaning that if a colleague joins or leaves our company you have to manage yet another account system.

Workflow integration in our JIRA platform is suboptimal. That is: a developer has to copy-paste links to the SOUP tool and that’s understandably a burden to our users. One alternative option is to automate this process using technology available in Jira and Confluence as suggested by my one of my colleagues.

#### Concluding Remarks

For now we have a system in place that fulfills our requirements and is actively used in Portavita. There are some disadvantages but at least we believe the tools help us to improve software security and quality.

#### Resources

[1] https://www.owasp.org/index.php/OWASP_Dependency_Check

[2] https://en.wikipedia.org/wiki/Software_of_unknown_pedigree

[3] https://wiki.jenkins.io/display/JENKINS/OWASP+Dependency-Check+Plugin

[4] https://www.archive.ece.cmu.edu/~ganger/712.fall02/papers/p761-thompson.pdf

[5] http://www.team-nb.org/wp-content/uploads/2015/05/documents2013/FAQ_62304_Final_130804.pdf
