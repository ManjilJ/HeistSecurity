# GitHub Security in 2026: From a Compromised Developer Extension to the Gheist Campaign

**A developer-oriented case study of software supply-chain compromise,
stolen credentials, GitHub Actions, and CI/CD security**

**Last updated:** October 27, 2026\
**Audience:** JavaScript/React developers, open-source contributors,
repository maintainers, and developers learning application security\
**Level:** Intermediate to advanced

------------------------------------------------------------------------

## Table of Contents

1.  [Introduction](#1-introduction)
2.  [The May 2026 GitHub Internal Repository
    Breach](#2-the-may-2026-github-internal-repository-breach)
3.  [Understanding the Attack Chain](#3-understanding-the-attack-chain)
4.  [The Gheist Campaign: A Different Attack, a Related
    Lesson](#4-the-gheist-campaign-a-different-attack-a-related-lesson)
5.  [How Gheist Abused GitHub
    Actions](#5-how-gheist-abused-github-actions)
6.  [Why GitHub's Secret Masking Was Not
    Enough](#6-why-githubs-secret-masking-was-not-enough)
7.  [Public Repositories Versus Private
    Repositories](#7-public-repositories-versus-private-repositories)
8.  [The Wider Risk: From GitHub to Cloud
    Infrastructure](#8-the-wider-risk-from-github-to-cloud-infrastructure)
9.  [What JavaScript and React Developers Should
    Learn](#9-what-javascript-and-react-developers-should-learn)
10. [Practical Repository Security
    Checklist](#10-practical-repository-security-checklist)
11. [What These Incidents Do Not
    Prove](#11-what-these-incidents-do-not-prove)
12. [Conclusion](#12-conclusion)
13. [References](#13-references)
14. [Disclaimers](#14-disclaimers)

------------------------------------------------------------------------

## 1. Introduction

GitHub is more than a website on which developers store source code.
Modern development workflows also use it to execute automated tests,
build applications, publish packages, deploy software, and access
credentials needed by external services.

This makes GitHub part of a larger system of trust involving developer
machines, code editors, third-party dependencies, authentication tokens,
automation workflows, and cloud infrastructure.

Two security developments reported in 2026 illustrate different ways
this system can be abused:

-   **May 2026:** A malicious release of the Nx Console Visual Studio
    Code extension contributed to a compromise that resulted in the
    exfiltration of approximately 3,800 internal GitHub repositories.
-   **September--October 2026:** A campaign tracked by Wiz Research
    under the name *Gheist* abused stolen GitHub access tokens to
    introduce malicious GitHub Actions workflows and attempt to extract
    secrets from repositories.

These were separate incidents. There is no established evidence that
they were conducted by the same threat actors.

Their technical connection is more general: once an attacker obtains
credentials belonging to a legitimate developer or automation account,
the attacker may be able to misuse the permissions already granted to
that identity.

Understanding these incidents requires no assumption that GitHub itself
was completely breached. It requires understanding how trust,
credentials, permissions, and automated code execution interact.

## 2. The May 2026 GitHub Internal Repository Breach

On May 18, 2026, GitHub detected unauthorized access to internal systems
originating from a compromised employee device.

The incident was facilitated by a maliciously modified release of Nx
Console, a Visual Studio Code extension. The affected version was
**18.95.0**.

GitHub's internal repositories were the target of the resulting
intrusion. Approximately 3,800 repositories containing proprietary
source code and internal configuration information were reportedly
exfiltrated.

GitHub's internal repositories should not be confused with customer
repositories. The incident reporting did not establish that all GitHub
customer repositories were compromised.

### 2.1 The supply-chain compromise preceding the breach

The attack did not begin with someone directly breaking into GitHub's
web servers.

The earlier entry point involved a compromise affecting TanStack-related
npm packages. According to the Nx postmortem, malicious package code
executed in a contributor's development environment and stole a GitHub
CLI OAuth token.

That credential was subsequently used to access the contributor's
repositories and publish a malicious extension release.

This is an example of a **software supply-chain attack**: instead of
attacking every downstream target directly, an attacker compromises a
trusted component or publishing process that other developers depend on.

### 2.2 The attack chain

``` text
Earlier npm supply-chain compromise
                |
                v
Malicious dependency executes
in a developer environment
                |
                v
GitHub CLI OAuth token stolen
                |
                v
Contributor's repository access abused
                |
                v
Malicious Nx Console v18.95.0 published
                |
                v
GitHub employee's development machine
runs the malicious extension
                |
                v
GitHub credentials and internal access exposed
                |
                v
Approximately 3,800 internal repositories
exfiltrated
```

The important point is that the attacker crossed several trust
boundaries. A compromised dependency affected a developer's machine; the
compromised developer's credentials enabled access to a publishing
process; the malicious release then reached another developer's
environment.

Each stage made the next stage possible.

### 2.3 Why the extension mattered

A VS Code extension is executable software, not merely a collection of
editor settings.

Extensions may run code on the developer's machine and may have access
to files, environment variables, and other resources permitted by the
operating system and the extension host.

The actual exposure depends on the extension's behavior, the platform,
and the permissions and credentials available in the environment.

In this incident, the malicious Nx Console release contained
credential-stealing functionality. An employee who received and
activated the affected version could therefore expose credentials
available to that environment.

The malicious version was distributed for a limited period on the Visual
Studio Marketplace and Open VSX. Nx advised anyone who installed the
affected version during the exposure window to treat the machine as
compromised and rotate exposed credentials.

### 2.4 What was compromised?

It is important to distinguish between three different things:

  -----------------------------------------------------------------------
  Item                                Meaning
  ----------------------------------- -----------------------------------
  Developer device                    The local environment in which the
                                      malicious extension executed

  Credentials                         Authentication material that could
                                      grant access to other systems

  Internal repositories               Source code and configuration data
                                      accessed and copied by the attacker
  -----------------------------------------------------------------------

The reported exfiltration of approximately 3,800 internal repositories
does not mean that 3,800 customer repositories were necessarily
breached. The Canadian Centre for Cyber Security stated that no action
was required for GitHub Enterprise Cloud customers in response to this
specific incident, while advising GitHub Enterprise Server customers to
follow the vendor's recommendations.

That distinction matters when interpreting headlines about a platform
breach.

## 3. Understanding the Attack Chain

Several familiar concepts from JavaScript development help explain why
this attack succeeded.

### 3.1 Dependencies execute code

Consider a normal Node.js project:

``` json
{
  "scripts": {
    "build": "vite build"
  },
  "dependencies": {
    "example-library": "^1.0.0"
  }
}
```

When a developer installs dependencies, package-manager behavior may
execute lifecycle scripts supplied by packages. Such scripts are a
legitimate part of the npm ecosystem, but they also create an
opportunity for malicious code to execute during installation.

A dependency does not have to be imported directly into a React
component to create risk. Code can execute before the application even
starts.

For example, a compromised dependency could attempt to read accessible
environment variables, local configuration files, or authentication
material. Whether it succeeds depends on the machine and its security
controls.

### 3.2 A valid token can be more useful than a password

A GitHub access token is a credential that allows an application or user
to authenticate to GitHub.

Depending on the token type, scopes, permissions, and organizational
policies, it may permit actions such as reading repository contents,
pushing commits, or interacting with automation workflows.

A stolen token may be usable without repeating the original interactive
login or multi-factor authentication process. The exact behavior depends
on the credential and the controls applied to it.

This is why the security of a token matters as much as the security of
the account that created it.

### 3.3 Trust can propagate through software

A useful way to understand the incident is to think of a chain of
dependencies and permissions:

-   A package is trusted by a developer.
-   The developer's environment has access to a repository.
-   The repository has permission to publish an extension.
-   Another developer trusts and installs that extension.
-   The second developer's environment has its own credentials and
    access.

An attacker who compromises one link may be able to exploit the trust
relationships beyond it.

The problem is not that every component is inherently unsafe. The
problem is that a single compromised component can inherit the
opportunities created by the surrounding system.

## 4. The Gheist Campaign: A Different Attack, a Related Lesson

Wiz Research reported tracking the Gheist campaign from September 15,
2026.

Unlike the May incident, Gheist did not depend on publishing a malicious
VS Code extension to enter a victim's environment. It used
**already-compromised GitHub access tokens** to create branches and
introduce malicious GitHub Actions workflows in repositories those
tokens could access.

The campaign illustrates what an attacker can do after acquiring a
credential with repository write access.

### 4.1 Reported scale

In its October 25 report, Wiz Research stated that it had recovered:

-   More than **13,500 malicious branches**.
-   More than **850 compromised GitHub accounts**.
-   Evidence involving personal projects and private enterprise
    repositories.
-   Successful workflow execution in private repositories and GitHub
    Enterprise environments.

These figures describe the research team's recovered evidence. They do
not mean that every branch successfully exposed secrets or that every
affected account represented a separate company.

The campaign was reported to be ongoing and evolving as of October 24.

### 4.2 The attack chain

``` text
GitHub access token stolen earlier
                |
                v
Repository write access obtained
                |
                v
Attacker creates a new branch
                |
                v
Malicious GitHub Actions workflow added
                |
                v
Workflow runs following a push
                |
                v
Accessible secrets written to workflow logs
in encoded form
                |
                v
Attacker retrieves the logs through GitHub's API
                |
                v
Workflow run deleted to reduce visible evidence
                |
                v
Exposed credentials may enable
further access to connected services
```

This attack is particularly instructive because the attacker used
legitimate GitHub features to carry out malicious actions.

The existence of a legitimate API, workflow engine, or automation
service does not make every use of it legitimate.

### 4.3 How to recognize the reported campaign

Wiz documented several indicators associated with Gheist:

  ------------------------------------------------------------------------
  Indicator                           Reported characteristic
  ----------------------------------- ------------------------------------
  Branch names                        `ci-sync-` followed by six
                                      characters

  Workflow path                       `.github/workflows/sync-check.yml`

  Commit message                      `ci: sync config`

  Workflow name                       `sync-check`

  API user-agent                      `gheist`

  Execution history                   Runs completed and were deleted
                                      shortly afterwards
  ------------------------------------------------------------------------

These indicators can help repository owners investigate suspicious
activity. They should not be treated as a universal signature for all
credential theft, nor does finding one indicator automatically prove
compromise.

## 5. How Gheist Abused GitHub Actions

For JavaScript developers, GitHub Actions can be understood as an
automation environment attached to a repository.

A workflow is typically defined in a YAML file under:

``` text
.github/workflows/
```

A project might use a workflow to install dependencies, run tests, build
a React application, or deploy a production release.

A simplified legitimate workflow could look like this:

``` yaml
name: React CI

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22

      - run: npm ci

      - run: npm run build
```

This workflow checks out the project, sets up Node.js, installs
dependencies, and builds the application.

The important detail is that a workflow is executable automation.
Changes to a workflow can change what code runs, what environment it
runs in, and what credentials become available to that execution.

### 5.1 The role of workflow permissions

GitHub Actions workflows can receive permissions through the
automatically provided `GITHUB_TOKEN` and may have access to other
configured secrets.

The `permissions` setting can restrict what the workflow's
`GITHUB_TOKEN` is allowed to do.

However, **restricting the workflow token is not the same as removing
every secret from the workflow's environment**.

Gheist exploited this distinction. According to Wiz, the malicious
workflow restricted its automatically provided GitHub token while
attempting to print secrets available through the workflow's secrets
context.

The attack therefore demonstrated why reviewing only the `GITHUB_TOKEN`
permissions is insufficient.

### 5.2 Repository secrets and environment secrets

GitHub Actions supports different categories of secrets, including
repository-level, organization-level, and deployment-environment
secrets.

These may contain credentials such as:

-   Cloud access credentials.
-   Package registry tokens.
-   Deployment keys.
-   API keys for external services.
-   Credentials used by release or publishing automation.

The exact secrets available to a workflow depend on its context, access
rules, and the relevant configuration.

A malicious workflow does not automatically obtain every secret
belonging to an organization. However, it may expose secrets that the
workflow is authorized to access.

This is why secret access should be limited to the workflows and
deployment environments that actually need it.

### 5.3 Why workflow changes deserve code-review attention

Developers often focus their review on application code such as React
components, Express routes, API handlers, and database operations.

But a change to a workflow can be equally important from a security
perspective.

For example, an unexpected change under `.github/workflows/` might:

-   Execute a new script.
-   Change which branches trigger a build.
-   Expand the permissions granted to automation.
-   Expose credentials to a job that previously did not need them.
-   Change the environment in which a deployment runs.

A repository review that ignores workflow files is incomplete.

## 6. Why GitHub's Secret Masking Was Not Enough

GitHub Actions can mask configured secret values in workflow logs. This
helps prevent accidental disclosure when a secret appears in its
original form.

However, masking is not the same as encryption, and it does not
guarantee that every transformed representation of a secret will be
recognized.

### 6.1 A simple illustration

Suppose an application uses an API key stored in an environment
variable.

The value might be:

``` text
EXAMPLE_SECRET_VALUE
```

A logging system may recognize that exact string and replace it with a
masked value.

But if the same value is transformed into another representation, the
result is a different sequence of characters.

The general relationship is:

``` text
Original secret
      |
      v
Transformation or encoding
      |
      v
Different text representation
      |
      v
Exact-value masking may not recognize it
```

This is the weakness exploited by the reported Gheist workflow. Wiz
documented the use of Base32 encoding to make secret values appear in
the logs without matching the original secret strings used by GitHub's
masking mechanism.

Base32 is an encoding scheme, not encryption. A person who obtains the
encoded data can decode it without knowing a secret key.

### 6.2 The broader lesson

Secret masking is a useful protective measure, but it is a final layer
of defense rather than a substitute for controlling secret access.

The stronger design is to ensure that a workflow never receives a secret
it does not need.

If an untrusted workflow cannot access a production credential, it
cannot expose that credential through its logs---even if log masking
fails.

## 7. Public Repositories Versus Private Repositories

One of the most important findings in the Gheist reporting concerns the
different protections applicable to public and private repositories.

Wiz reported that GitHub had introduced a control on July 28, 2026, that
holds certain potentially malicious workflow runs in public repositories
for collaborator approval.

The researchers reported that the public-repository Gheist runs they
examined were held for approval and did not execute.

However, the observed campaign successfully executed workflows in
private repositories and GitHub Enterprise environments where that hold
did not apply in the same way.

This is a significant distinction, but it should be interpreted
carefully.

It does not mean that all public repositories are secure or that all
private repositories are vulnerable. Other attack paths and repository
configurations can change the outcome.

The broader principle is that **security controls must be evaluated
against the exact environment and permissions in which an action runs**.

Repository visibility is only one factor. Workflow approval rules, token
permissions, branch protections, environment restrictions, and runner
configuration all contribute to the security boundary.

## 8. The Wider Risk: From GitHub to Cloud Infrastructure

A compromised GitHub credential may have consequences beyond the
repository itself.

Many applications use GitHub Actions to deploy software to cloud
services. Workflows may authenticate to AWS, Azure, Google Cloud, or
other platforms.

If a workflow has access to a cloud credential, that credential may
provide access to resources outside GitHub.

This creates a possible chain of exposure:

``` text
GitHub token compromised
          |
          v
Repository modified
          |
          v
Workflow executed
          |
          v
Available secret exposed
          |
          v
Cloud credential obtained
          |
          v
Additional cloud resources accessed
```

This chain is not automatic. Every step depends on the permissions,
configuration, and credentials involved.

Nevertheless, Wiz reported follow-on activity in which cloud credentials
associated with affected organizations were used to enumerate AWS
Secrets Manager.

### 8.1 What is OpenID Connect in this context?

OpenID Connect (OIDC) is an identity protocol that can be used by GitHub
Actions to obtain short-lived credentials from a cloud provider.

Instead of storing a long-lived cloud access key directly as a GitHub
secret, an organization can configure a trust relationship in which an
eligible workflow requests an identity token and exchanges it for
temporary cloud credentials.

This can reduce the need to store persistent cloud keys.

However, OIDC does not automatically make a deployment secure. If a
cloud trust policy accepts tokens from untrusted branches or workflows,
an attacker may still be able to obtain credentials through an
improperly configured trust relationship.

A secure implementation should restrict which repository, branch,
environment, or workflow identity is allowed to assume a cloud role.

### 8.2 Why self-hosted runners require additional care

GitHub Actions can run on GitHub-hosted machines or on self-hosted
runners managed by the repository owner.

A self-hosted runner may have access to local files, internal networks,
cloud metadata services, container runtimes, or credentials used by
other automation.

That makes its security boundary different from that of a disposable
hosted runner.

Wiz reported that a later Gheist variant attempted to target self-hosted
runners and search for additional credentials. The report also described
defects in that variant, so the attempted capabilities should not be
mistaken for proof that every such operation succeeded.

The practical lesson is to treat a runner as a machine executing
code---not as a harmless extension of the GitHub interface.

## 9. What JavaScript and React Developers Should Learn

These incidents have direct relevance to developers who build
applications using JavaScript, React, Node.js, and related tools.

### 9.1 A repository contains more than application source code

A typical React project includes:

-   Application components.
-   Dependencies and package-manager configuration.
-   Build scripts.
-   Environment configuration.
-   CI/CD workflows.
-   Deployment scripts.
-   Sometimes infrastructure configuration.

Each of these may influence how code is installed, executed, built, or
deployed.

Security review should therefore include the complete development
workflow, not only the source files that render the user interface.

### 9.2 `package.json` deserves attention

Consider this section of a package manifest:

``` json
{
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "prepare": "node scripts/prepare.js"
  }
}
```

The `prepare` script is part of the package lifecycle. Depending on how
the package is installed or used, it may execute code automatically.

A developer who runs `npm install` or `npm ci` is not merely downloading
inert files in every possible configuration. Package lifecycle behavior
and the scripts supplied by dependencies must also be considered.

This does not mean that lifecycle scripts are inherently malicious. It
means that package installation is a potential code-execution boundary.

Practical precautions include reviewing unfamiliar dependencies, keeping
package managers and dependencies updated, checking unexpected lifecycle
scripts, and using lockfiles consistently.

### 9.3 Environment variables are not automatically safe

JavaScript developers commonly use environment variables for API keys
and deployment configuration.

For example:

``` javascript
const apiKey = process.env.API_KEY;
```

In Node.js, this reads an environment variable available to the running
process. It does not encrypt the value or make it inaccessible to other
code executing with sufficient access.

Similarly, storing a value in a `.env` file does not itself protect it.

In frontend applications, build tools may also embed selected
environment values into the generated client bundle. A secret included
in browser-delivered JavaScript should be considered exposed to users
who can inspect that bundle.

Server-side credentials should remain on the server or within
appropriately restricted deployment environments.

### 9.4 A GitHub Actions workflow is code

A workflow may be written in YAML rather than JavaScript, but it can
execute shell commands, invoke Node.js scripts, install packages, and
interact with external services.

It should be reviewed with the same seriousness as application code that
can access sensitive resources.

In particular, unexpected changes to `.github/workflows/`, package
lifecycle scripts, build configuration, and deployment scripts deserve
attention.

### 9.5 Authentication is not the same as authorization

Authentication answers: *Who or what is making this request?*

Authorization answers: *What is that identity allowed to do?*

A stolen GitHub token may pass authentication successfully because
GitHub recognizes the token as valid. The remaining security question is
what permissions that token grants.

This is why narrowly scoped tokens, short lifetimes, branch protections,
environment approvals, and least-privilege access are important.

Multi-factor authentication helps protect interactive account access,
but it does not make an already-stolen access token harmless.

### 9.6 Open-source does not mean risk-free

Public source code makes review possible, but a public repository can
still receive a malicious commit or dependency update.

A legitimate publisher account may be compromised. A package can be
modified after developers have begun trusting it. An apparently normal
workflow can execute an unexpected script.

Open-source security therefore involves both inspecting code and
maintaining control over the identities, dependencies, and automation
that deliver it.

## 10. Practical Repository Security Checklist

The following checklist is intended for an individual JavaScript/React
developer maintaining projects on GitHub. It is not a substitute for an
organization's full security assessment.

### Account and credential hygiene

-   [ ] Enable two-factor authentication on the GitHub account.
-   [ ] Review personal access tokens and revoke those that are
    unnecessary or no longer used.
-   [ ] Use narrowly scoped, fine-grained tokens where appropriate.
-   [ ] Never commit passwords, private keys, API keys, or real `.env`
    files.
-   [ ] If a secret was committed, remove it from the current code and
    rotate the exposed credential. Deleting the file alone is
    insufficient.
-   [ ] Review account sessions, authorized applications, and recent
    security activity when investigating suspicious behavior.

### Repository integrity

-   [ ] Review unexpected commits, branches, tags, and
    repository-setting changes.
-   [ ] Inspect `.github/workflows/` when reviewing unfamiliar
    repository changes.
-   [ ] Investigate unexpected changes to `package.json`, lockfiles,
    build scripts, and deployment configuration.
-   [ ] Keep branch protection and pull-request review enabled where
    appropriate.
-   [ ] Avoid granting write access to identities that need only read
    access.

### GitHub Actions

-   [ ] Set the minimum required workflow permissions.
-   [ ] Do not expose production secrets to workflows that do not
    require them.
-   [ ] Use environment protection rules and required reviewers for
    sensitive deployments.
-   [ ] Pin third-party actions to verified commit SHAs when
    appropriate, rather than relying solely on mutable version tags.
-   [ ] Review workflow triggers carefully, particularly those involving
    pull requests and untrusted contributions.
-   [ ] Avoid running untrusted code in privileged workflows.
-   [ ] Restrict cloud OIDC trust policies to the intended repository
    and deployment context.

### Dependency and extension hygiene

-   [ ] Install VS Code extensions from trusted sources and review
    publisher identity and release information.
-   [ ] Keep extensions and package managers updated, while recognizing
    that updates themselves can occasionally introduce supply-chain
    risk.
-   [ ] Review unexpected dependency changes and installation scripts.
-   [ ] Commit and maintain the appropriate package-manager lockfile.
-   [ ] Investigate suspicious package releases or unexpected network
    activity during installation.

### Investigating Gheist indicators

If a repository shows signs associated with the reported Gheist
campaign:

-   [ ] Look for unexpected branches matching the reported `ci-sync-`
    naming pattern.
-   [ ] Inspect suspicious `sync-check.yml` workflow additions and
    associated commits.
-   [ ] Review available Actions history and repository audit logs for
    completed and subsequently deleted runs.
-   [ ] Investigate suspicious activity associated with the `gheist`
    user-agent indicator.
-   [ ] If a workflow may have executed, identify the repository,
    organization, and environment secrets it could access.
-   [ ] Revoke compromised tokens and rotate potentially exposed
    secrets.
-   [ ] Investigate connected cloud accounts and self-hosted runners
    where applicable.

Do not conclude that a repository is safe simply because these
indicators are absent. Indicators are useful for detecting known
activity, not for proving the absence of every possible attack.

## 11. What These Incidents Do Not Prove

A responsible case study should distinguish established findings from
broader conclusions.

**The May incident does not establish that all GitHub customer
repositories were compromised.** The reported exfiltration involved
GitHub's internal repositories.

**Gheist does not establish that GitHub's core infrastructure was
breached.** The reported campaign abused stolen credentials and
legitimate repository automation.

**The two incidents are not established to have the same perpetrators.**
Similar techniques do not prove common attribution.

**The reported branch count is not a count of confirmed secret
disclosures.** A malicious branch may be planted without its workflow
successfully executing or extracting usable credentials.

**The existence of an API key in a repository does not automatically
prove that it was abused.** Exposure creates risk, but evidence is
needed to establish subsequent unauthorized use.

**A public repository is not automatically safe, and a private
repository is not automatically compromised.** Risk depends on the
available credentials, permissions, workflow configuration, and security
controls.

These distinctions are important because security reporting can
otherwise turn a documented incident into unsupported generalizations.

## 12. Conclusion

The May 2026 GitHub internal repository breach and the subsequent Gheist
campaign illustrate two different stages of the modern software
supply-chain threat.

In May, a malicious dependency and a compromised developer credential
contributed to the distribution of a trojanized editor extension,
ultimately exposing GitHub's internal repositories.

Gheist demonstrated how previously stolen GitHub credentials could be
reused to introduce malicious workflows, attempt to extract secrets
through workflow logs, and potentially extend access into connected
infrastructure.

For JavaScript and React developers, the lesson is practical rather than
abstract.

The code we write is only one part of the software we deliver.
Dependencies, editor extensions, build scripts, workflow files, access
tokens, environment variables, and deployment credentials all
participate in the process.

A React application may be perfectly correct at the component level
while its development or deployment pipeline remains vulnerable.

Security is therefore not limited to preventing bugs in application
code. It also involves controlling what code executes, which identities
can execute it, what credentials are available, and what those
credentials can access.

The most useful approach is not to distrust every tool or avoid
automation. It is to make trust explicit, permissions narrow,
credentials replaceable, and unexpected changes visible.

## 13. References

The following sources provide the primary reporting and technical
details used in this case study.

1.  **Nx --- Postmortem: Nx Console v18.95.0 Supply-Chain Compromise**\
    https://nx.dev/blog/nx-console-v18-95-0-postmortem\
    Primary source for the malicious extension release, the earlier
    TanStack compromise, the stolen contributor credential, the exposure
    window, and remediation.

2.  **Canadian Centre for Cyber Security --- AL26-013: Security Incident
    Impacting GitHub Internal Repositories**\
    https://www.cyber.gc.ca/en/alerts-advisories/al26-013-security-incident-impacting-github-internal-repositories\
    Government security advisory covering the May incident and
    recommended actions.

3.  **Wiz Research --- Gheist: Attempted Scaled Abuse of Stolen GitHub
    Tokens Stumbles**\
    https://www.wiz.io/blog/gheist-attempted-scaled-abuse-of-stolen-github-tokens-stumbles\
    Primary research report for the Gheist campaign, its reported scale,
    attack mechanics, indicators, and observed impact.

4.  **GitHub Actions documentation --- Controlling permissions for
    `GITHUB_TOKEN`**\
    https://docs.github.com/en/actions/security-for-github-actions/security-guides/automatic-token-authentication\
    Background on workflow-token permissions and how to restrict access.

5.  **GitHub Actions documentation --- Using secrets in GitHub
    Actions**\
    https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions\
    Background on repository, organization, and environment secrets.

6.  **GitHub Actions documentation --- Security hardening for GitHub
    Actions**\
    https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions\
    Guidance on workflow security, third-party actions, permissions, and
    untrusted code.

7.  **GitHub documentation --- OpenID Connect**\
    https://docs.github.com/en/actions/concepts/security/openid-connect\
    Background on using OIDC to obtain cloud credentials without storing
    long-lived cloud credentials directly in GitHub.

## 14. Disclaimers

### Scope and purpose

This document is an educational, developer-oriented case study intended
to explain reported security incidents and their relevance to everyday
software development. It is not a formal security audit,
penetration-test report, incident-response report, or professional
security assessment.

### Source and attribution

The incident descriptions and campaign statistics are based on the cited
public reporting available at the stated update date. Where the document
discusses broader risks or architectural lessons, those discussions are
analysis rather than claims that the same behavior occurred in every
affected environment.

No attribution linking the May GitHub incident and Gheist to the same
threat actor is asserted.

### Accuracy and limitations

Security investigations evolve. Technical findings, reported numbers,
indicators of compromise, and vendor guidance may change as further
evidence becomes available. Readers should consult the original sources
for the latest findings and verify indicators against their own
environments.

The presence or absence of a reported indicator does not independently
establish whether a system has been compromised.

### No guarantee of security

The checklist and examples are general guidance, not a guarantee that a
repository, development machine, or deployment environment is secure.
Security requirements depend on the application's architecture,
deployment model, credentials, permissions, and threat environment.

### Responsible use

The technical material is presented for defensive understanding,
software maintenance, and security awareness. Any investigation of
repositories, credentials, or infrastructure should be conducted only on
systems the investigator owns or is explicitly authorized to examine.

### Authorship and transparency

This document is a synthesized educational analysis based on publicly
available sources. It should not be represented as original forensic
research, direct observation of the affected systems, or an official
statement from GitHub, Nx, Wiz, or any other cited organization.

Corrections and updates should be made when authoritative sources
provide material new information.
