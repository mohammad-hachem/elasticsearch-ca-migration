# Elasticsearch Migration Without Enrollment Tokens
## Node replacement, CA rotation, and client trust

I worked on an Elasticsearch migration where the cluster had started on 7.x and was already running 8.17.0. It still had manually configured security, so the enrollment-token workflow was not the right way to add replacement nodes.

The important part was making old and new nodes trust each other during the transition. I introduced new certificate material and worked through that trust boundary before progressing with node replacement. When the first replacement joined, the cluster returned to green. That was a useful checkpoint, but it did not establish that every client had moved or that old trust could be removed.

This guide turns that experience into a staged procedure with fictional names, technical examples, validation gates, and rollback decisions.

**[Read the complete migration guide →](MIGRATION_GUIDE.md)**

### What it covers

- Why a manually secured cluster needs manual node configuration.
- Transport TLS versus HTTP TLS.
- Certificate inspection, separate CAs, per-node certificates, and overlapping trust.
- Existing-node changes, replacement-node configuration, and rolling restarts.
- Shard evacuation, watermarks, allocation awareness, and master quorum.
- Kibana and Kafka Connect client trust and endpoint cutover.
- Expired legacy certificates, troubleshooting, rollback, and removing old trust.

### The rule I would keep in front of me

**Distribute trust before presenting a new identity. Retire old trust only after its dependencies are gone.**

### Scope

The case originated in the Elasticsearch 8.17.0 era. The examples target that configuration model; 8.17 documentation is now archived. This is not a recommendation to install an old release today or to mix arbitrary versions in a cluster.

This procedure replaces nodes within the **same cluster**. A separate-cluster migration or major-version upgrade needs a different plan. The command examples are reconstructed for publication, not a replay of production logs or a claim that this exact example was executed end to end.

All hostnames, addresses, and filenames are illustrative. No employer or customer details, credentials, private keys, or production configurations are included.

For the broader certificate-lifecycle design, see [Internal PKI and Machine Identity](https://github.com/mohammad-hachem/internal-pki-machine-identity).
