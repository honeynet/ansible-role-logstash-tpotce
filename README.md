# Ansible Role: Logstash

Logstash 9.x running the T-Pot honeypot pipeline, shipping to an authenticated,
TLS-protected Elasticsearch cluster.

## Provenance

This role is a fork with two upstreams, and syncs should be taken from both:

| Part | Upstream | Synced at |
| --- | --- | --- |
| Role scaffolding (`handlers/`, `tasks/ssl.yml`, `tasks/plugins.yml`) | [geerlingguy/ansible-role-logstash](https://github.com/geerlingguy/ansible-role-logstash) | `c8aaeb69` |
| Pipeline configuration (`files/filters/`, `templates/outputs/`) | [telekom-security/tpotce](https://github.com/telekom-security/tpotce), `docker/elk/logstash/dist/` | `8a228130` |

Files taken verbatim from geerlingguy are left formatted as upstream has them
(short-form module calls, unqualified names) so that the next sync stays a clean
diff. The repository `.ansible-lint` skips the rules that would otherwise
object.

## How this diverges from T-Pot

T-Pot runs one monolithic `logstash.conf` keyed on `[type]`, tails honeypot logs
off local disk with `file` inputs, and writes everything into a single
`logstash-YYYY.MM.dd` index.

This deployment is a *hive*: sensors ship over **beats** on 5044, events carry
lowercase **tags** rather than a `[type]`, and each honeypot gets its own index
(`logstash-cowrie-2026.08.24`). That split is driven by `index_list` in
`vars/main.yml`, rendered through `templates/outputs/tpot.output.conf.j2`.

Both divergences predate this sync and are preserved deliberately — changing
either would invalidate the existing Kibana saved objects.

## Changes required for 9.x

These were hard breaks, not deprecations:

* **`logstash-filter-translate`** renamed `field` → `source` and `destination`
  → `target` in 4.0. The old names are removed. Affected
  `100_suricata.filter.conf` and the GeoIP enrichment filter.
* **`logstash-input-beats` 7.0** made `ssl` and `ssl_verify_mode` obsolete;
  they are now `ssl_enabled` and `ssl_client_authentication`. An obsolete
  setting fails the pipeline at startup rather than warning.
* **`logstash-output-elasticsearch` 12.0** made `cacert`, `ssl` and
  `ssl_certificate_verification` obsolete, in favour of
  `ssl_certificate_authorities`, `ssl_enabled` and `ssl_verification_mode`.
* **`pipeline.ecs_compatibility`** defaulted to `disabled` up to 7.x and to
  `v8` from 8.0. The entire T-Pot filter set uses non-ECS field names
  (`src_ip`, `dest_port`), so this role pins it back to `disabled` in both
  `logstash.yml` and `pipelines.yml`, exactly as T-Pot does.
* **GeoIP databases** no longer live under
  `/usr/share/logstash/vendor/bundle/jruby/`. 8.x moved them into the GeoIP
  database management service. The old role searched that path with `find` and
  fed `files[0].path` into `set_fact`, which on 9.x fails on an empty list. The
  filter now uses `default_database_type => "City"` / `"ASN"` against the
  bundled Creative Commons databases, matching T-Pot.

## Honeypot coverage

The filter set was brought up to what T-Pot emits today. Added: `beelzebub`,
`ddospot`, `endlessh`, `galah`, `go-pot`, `h0neytr4p`, `hellpot`, `honeyaml`,
`honeypots`, `miniprint`, `nginx`, `rdphoneypot`, `redishoneypot`,
`sentrypeer`, `wordpot`.

Kept, although T-Pot has since dropped them, because sensors running older
T-Pot releases may still send them: `honeypy`, `honeysap`, `rdpy`.

> The tag names for the newly added honeypots follow this repository's existing
> convention — T-Pot's log type name, lowercased. **Check them against the
> filebeat configuration on your sensors.** A filter whose tag never matches is
> inert rather than harmful, but it also does nothing.

## Credentials

The Elasticsearch username and password go into the **Logstash keystore** as
`ES_USER` and `ES_PASSWORD`, referenced from the output configuration, so no
credential is written into `/etc/logstash/conf.d`. Keystore entries are only
written when absent; after rotating the password, run once with
`-e logstash_force_keystore_update=true`.

## Listbot

`logstash_manage_listbot` downloads the CVE and IP reputation translation maps
from `listbot.sicherheitstacho.eu` into `/etc/listbot`. These feed the
`translate` filters in `100_suricata` and `198_geoipenrich`. Set it to `false`
if that host is unreachable; the filters then simply do not enrich.

## Debian only

`tasks/setup-RedHat.yml` and `templates/logstash.repo.j2` have been dropped.
The deployment targets Debian 13, the version pinning and repository handling
are Debian-specific, and an untested RedHat path is worse than an explicit
failure. The role now fails fast on a non-Debian `os_family`.

## Molecule

`molecule test` converges the role on `geerlingguy/docker-debian13-ansible`,
generates a throwaway CA and cluster credentials on the controller, and
verifies the compiled pipeline, the keystore entries, the rendered output
config and that Logstash starts clean. It replaces the old scenario, which
converged `geerlingguy.java` and `geerlingguy.elasticsearch` — neither of
which this role depends on any more.

## Example Playbook

    - hosts: logstash
      roles:
        - role: honeynet.logstash
      vars:
        elastic_version: "9.3.5"
        elastic_repo_channel: "9.x"
        logstash_elasticsearch_hosts:
          - https://elasticsearch-01.example.org:9200
        logstash_elasticsearch_ca_file: /etc/logstash/certs/ca.crt

See `defaults/main.yml` for the full set of variables, and
`molecule/default/converge.yml` for a complete working configuration.

## License

MIT

## Author Information

The Honeynet Project. Role scaffolding originally by
[Jeff Geerling](https://github.com/geerlingguy).
