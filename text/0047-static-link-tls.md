- Feature Name: static_link_tls
- Start Date: 2022-02-04
- RFC PR: https://github.com/volta-cli/rfcs/pull/47
- Volta Issue: https://github.com/volta-cli/volta/pull/1214

# Summary
[summary]: #summary

Proposal to switch to a statically-linked TLS implementation for HTTPS downloads via the [`Rustls`](https://github.com/rustls/rustls) crate.

# Motivation
[motivation]: #motivation

Currently, we support HTTPS downloads using the `native-tls` feature of [`attohttpc`](https://github.com/sbstp/attohttpc). That uses a different implementation of TLS on each platform, most notably using dynamically-linked OpenSSL on Linux. Maintaining support for OpenSSL has been a significant burden in the past, as there are a number of different binary-incompatible versions that we need to link against to support the majority of Linux distros. Additionally, as noted in [this issue](https://github.com/volta-cli/volta/issues/1150), OpenSSL 3.0 is on the verge of wider adoption, so that would add an extra version that we need to maintain in our build pipeline.

Switching to use the `rustls` feature would instead use a statically-linked, Rust-based implementation of TLS which is supported out of the box on all of the platforms that we currently support. That would allow us to greatly simplify our build pipeline and install script, since we would no longer need to worry about the specific OpenSSL version available on the user's machine. It would also mean that we don't run into compatibility / install issues with future releases of OpenSSL.

# Pedagogy
[pedagogy]: #pedagogy

This change should be purely an implementation detail, so there shouldn't be any user-facing changes to document. We should make sure to update any references to OpenSSL in our documentation. We could also potentially add a reference section about the TLS implementation on which we are relying.

# Details
[details]: #details

Since this is purely a change to the underlying implementation in a transitive dependency, there shouldn't be any code changes necessary in the Volta binary itself. The implementation of this change will involve 3 steps:

1. Update our dependency on `attohttpc` to remove the default features and add `rustls` - This needs to happen in both crates that depend on `attohttpc`
2. Update our GitHub Actions CI workflow to skip the convoluted OpenSSL setup and instead build & publish a single binary for Linux
3. Update our install script to handle the fact that previous versions relied on OpenSSL but going forward there will only be a single binary

# Critique
[critique]: #critique

There are a number of potential issues to consider when switching to a statically-linked TLS implementation:

## Updating

One benefit of dynamically-linked libraries is that the user can update to a new version (perhaps to fix a security vulnerability) without needing to update the consuming app. So if there is a new (binary-compatible) version of OpenSSL released, users can install it on their machines and Volta will immediately benefit from any fixes. With a statically-linked TLS implementation, we lose that benefit. If there is a security issue with Rustls, we will need to bump the version ourselves and release a new Volta version that includes the fixes.

Staying on top of updates to Rustls will definitely add to the necessary maintenance of Volta. However, now that we have dependabot integrated and automatically generating warnings and PRs for new versions of our dependencies, I believe the extra cost will be fairly small. Additionally, Rustls appears to be fairly stable and hasn't had a flurry of releases where the burden of upgrading would be significant.

The other issue with updating is that we need to notify our users when there is a new security fix available. While we have existing channels to announce to users that there is a new Volta version available, we don't currently have an auto-update feature built-in. Combined with the fact that Volta is intended to be transparent and unobtrusive, many users may miss the fact that there is a new version entirely and not update. This is the biggest concern I have with switching to a statically-linked implementation.

## Security

OpenSSL, though it has more than its share of vulnerabilities, is a well-tested and widely-used library. As such, there's a minimum level of security expected from it. By contrast, Rustls is relatively new to the space and hasn't been battle-tested to nearly the same level. However, back in mid-2020, Rustls (and the underlying cryptography libraries on which it relies) got a security audit from a 3rd-party security firm. The [results](http://jbp.io/2020/06/14/rustls-audit.html) of that audit were very positive, noting the quality of the code and only locating a few minor issues that weren't immediately exploitable. While no software is perfectly secure, Rustls seems to be on very solid ground re: security.

## Binary size / compilation time

When switching to statically-linked TLS, there will be an impact on our binary size and compilation time. Anecdotally, from my local testing, the binaries appear to be ~20% larger using Rustls than using native TLS. For compile time, since it is a transitive dependency, there shouldn't be any impact on incremental compile times. For CI, though the actual time to compile Volta will be slightly longer, I suspect the overall impact will be a net positive. We won't need to do 3 parallel builds for Linux, nor will we need to manually compile OpenSSL for each, so the total time to prepare a production Linux build should drop significantly.

## Startup Latency

With the increased binary size, another possible concern is the impact on startup latency, especially for `volta-shim` which is on the hot path of launching a tool. However, in my highly non-scientific testing, I wasn't able to find any difference in the load time between the statically and dynamically linked versions of Volta. Running a large number of attempts, both had nearly the exact same mean load time with mostly overlapping confidence intervals, so I don't believe there will be a major impact on load time. That said, it would still be worthwhile to do some additional tests on different environments to make sure there aren't any obvious regressions.

# Unresolved questions
[unresolved]: #unresolved-questions

- Versioning: If we make the switch, should we increment the minor version (i.e. move to Volta 1.1)? Ultimately, since there shouldn't be any visible changes for users, it would be reasonable to make it only a patch version update. On the other hand, having a minor version marker makes it a little easier to disambiguate the difference (e.g. it's _slightly_ easier to remember that Volta 1.1 and above are statically-linked, rather than Volta 1.0.6 and above)
- Do we need to do any sort of announcement / messaging around the change in underlying implementation?
- Does the install script need to maintain the ability to install older versions of Volta, or can we remove the OpenSSL detection logic entirely?
