# Security Gates

A **security gate** is a checkpoint in the pipeline that decides whether code or an artifact is allowed to proceed — to merge, to be released, or to be deployed — based on security criteria. Gates are how shift-left scanning translates into actual risk reduction: a finding that never blocks anything rarely gets fixed.

The hard part is not adding gates; it is designing gates that reduce risk **without destroying developer flow**. Gates that are too noisy or too strict get disabled, ignored, or bypassed.

## Principles for effective gates

- **Gate on risk, not raw counts** — block on severity, exploitability, and reachability, not on the total number of findings. A gate that blocks on "any medium CVE in any transitive dependency" will be overridden within a week.
- **Baseline existing issues** — hold teams accountable for *new* risk they introduce, not the entire backlog of legacy debt at once. Use a known-good baseline snapshot and block only on findings that did not exist in the baseline.
- **Fail fast and clearly** — when a gate blocks, tell the developer exactly what, where, and how to fix it. A gate that says "build failed" with no guidance is friction without value.
- **Provide a path forward** — support documented, time-boxed risk acceptance / exceptions with an owner, rather than forcing developers to silently disable checks.
- **Tune relentlessly** — high false-positive rates are the fastest way to lose developer trust; treat noise as a bug in the gate, not a developer compliance problem.

## Where to place gates

| Stage | Typical gate | Failure mode |
| --- | --- | --- |
| Pre-commit / IDE | Secrets, linting, fast SAST (advisory) | Advisory only — educate, don't block here |
| Pull request | SAST, SCA, IaC — block new high/critical | Block merge; surface in PR comment with fix guidance |
| Build | Full scans, container scan, SBOM generation | Block artifact promotion from build stage |
| Release | Signed artifacts, provenance, no unresolved criticals | Block publishing to production registry |
| Deploy | Admission control: only signed, policy-compliant artifacts | Block workload from running in cluster |

```yaml
# Example: GitHub Actions gate using Semgrep — blocks on new high findings
- name: Semgrep SAST gate
  uses: returntocorp/semgrep-action@v1
  with:
    config: >
      p/owasp-top-ten
      p/secrets
    publishToken: ${{ secrets.SEMGREP_APP_TOKEN }}
  env:
    SEMGREP_BASELINE_REF: ${{ github.base_ref }}  # diff-aware: new findings only
```

## Exception and risk-acceptance process

No gate system survives contact with reality without an exception process. Without one, developers disable gates rather than deal with blocked pipelines. A sound process:

1. **Raise an exception request** — the developer (or team) documents the finding, explains why immediate remediation is not feasible (e.g., no fix available, legacy library), and proposes a compensating control.
2. **Approve with a risk owner** — a security engineer or application security lead reviews and approves; the approver takes ownership of the accepted risk.
3. **Time-box every exception** — set a hard expiry (e.g., 30 or 90 days). The gate re-engages automatically when the exception expires. Indefinite exceptions are not exceptions — they are hidden vulnerabilities.
4. **Track exceptions centrally** — maintain a register (in a vulnerability management tool or ticketing system) so accepted risks are visible to leadership, not hidden in `.semgrepignore` files.
5. **Review on a cadence** — include open exceptions in sprint planning and quarterly security reviews so they get addressed, not forgotten. Report exception counts and age to leadership as a risk indicator.

## Smarter prioritization

Mature programs increasingly drive gates from **correlated, contextual risk** rather than isolated scanner output. Application Security Posture Management ([ASPM](../../3-Governance/3-3-Reporting/3-3-3-ASPM.md)) aggregates findings across tools, deduplicates them, and adds code-to-runtime context so gates block on what is genuinely exploitable and reachable in production — keeping signal high and friction low.

The evolution of gate intelligence:

| Generation | What drives the gate |
|---|---|
| 1st | Raw count of findings above a severity threshold |
| 2nd | New findings since baseline, filtered by severity |
| 3rd | CVSS + EPSS + KEV — exploitability-weighted severity |
| 4th | Reachability + runtime exposure + ASPM correlation |

## Common pitfalls and anti-patterns

- **Gates without feedback** — a gate that blocks with no explanation sends developers to Google. Every blocked gate must link to the finding, the affected code, and remediation guidance.
- **Gates configured but not enforced** — required CI status checks must be enabled in branch protection rules; otherwise developers merge without them.
- **Exception process that requires security team approval for every finding** — creates a bottleneck and incentivizes teams to avoid scanning. Delegate tier-2 and tier-3 exception approvals to risk owners within the product team.
- **No visibility into exception trends** — if the exception register is growing every sprint, that is a systemic problem. Track exception counts, ages, and owners as board-level metrics.
- **Suppression via comments in source code** — `# nosec`, `// NOSONAR`, and `.semgrepignore` suppressions are invisible in most dashboards. Require that suppressions be tracked in the central exception register.

## Maturity progression

**Starter** — Enable SAST and SCA scans in CI. Run in advisory mode (report, don't block) for two sprints to establish baseline. Then block on new criticals.

**Intermediate** — Gate pull requests on new high/critical SAST and SCA findings. Enforce a formal exception process. Track exceptions in DefectDojo or Jira. Add container scan gates at the build stage.

**Advanced** — Drive gates from ASPM-correlated, reachability-enriched findings. Fully automate exception expiry. Measure false positive rate per scanner and tune quarterly. Report gate compliance and exception trends to engineering leadership monthly.

## Tools

| Category | Examples |
|---|---|
| Vulnerability tracking & gate integration | DefectDojo (open source), Archery |
| Policy-as-code gate enforcement | OPA/Conftest, Rego policies in CI, Kyverno (Kubernetes) |
| CI/CD native gates | GitHub branch protection + required status checks, GitLab merge request approvals, Jenkins Quality Gates |
| ASPM (correlated gate decisions) | Apiiro, Arnica, Ox Security, Cycode — see [ASPM](../../3-Governance/3-3-Reporting/3-3-3-ASPM.md) |
| Exception / risk-acceptance tracking | Jira (security issue type), ServiceNow, DefectDojo risk acceptance workflow |

## Metrics and KPIs

| Metric | Target |
|---|---|
| % of PRs passing security gates without exception | > 95% |
| Open exceptions older than 90 days | 0 |
| Gate false positive rate | < 15% per tool |
| Findings that escape to production | Decreasing quarter-over-quarter |
| Mean time from gate block to resolution | < 3 business days (critical) |

---

## Further reading

- [OWASP DSOMM — Test & Verification](https://dsomm.owasp.org/)
- [OWASP Top 10 CI/CD Security Risks](https://owasp.org/www-project-top-10-ci-cd-security-risks/)
- [NIST SSDF — PW & RV practices](https://csrc.nist.gov/Projects/ssdf)
- [DefectDojo documentation](https://defectdojo.github.io/django-DefectDojo/)
