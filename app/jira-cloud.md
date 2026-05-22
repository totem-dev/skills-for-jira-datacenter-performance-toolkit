# Cloud Run — Pre-flight checklist (what jira_prepare_data.py needs to exist)

Before running `bzt_on_pod.sh jira.yml`, the cloud Jira instance (provisioned from APT snapshot) needs:

## 1. Plugin installed
Follow `install-plugin.md`. TL;DR:
- Add `-Dupm.plugin.upload.enabled=true -Dupm.pac.enabled=false` to JVM flags (via dcapt.tfvars or kubectl exec)
- `kubectl cp skillsforjira-*.jar atlassian/jira-0:.../installed-plugins/`
- Restart pod

## 2. Base URL must match application_hostname
Jira's configured base URL (Admin > System > General Configuration) must match what APT uses.
- APT uses whatever is in `jira.yml` → `application_hostname`
- Check: `curl -su admin:admin http://<elb>/rest/api/2/serverInfo | python -m json.tool | grep baseUrl`
- If wrong: update via Admin UI or DB: `UPDATE propertystring SET propertyvalue='http://<elb>/jira' WHERE id=(SELECT id FROM propertyentry WHERE property_key='jira.baseurl')`
- If base URL doesn't match the `uri` parameter in `dashboard-diagnostics`, `locust_view_dashboard` will FAIL

## 3. "Worker Skillset" custom field
The plugin auto-creates this field on first startup. Verify it exists:
`curl -su admin:admin "http://<elb>/jira/rest/api/2/field" | python -c "import sys,json; fields=json.load(sys.stdin); print([f['name'] for f in fields if 'Worker Skillset' in f['name']])`
If missing: create via Admin > Issues > Custom Fields > Add (Skillset Field type).

## 4. Data preparation
`jira_prepare_data.py` runs automatically as part of `bzt jira.yml`. It validates:
- Kanban and scrum boards exist (APT snapshot has these)
- Performance test users exist (script creates them)
- Custom issues CSV populated (uses `custom_dataset_query` JQL)
The snapshot from Atlassian already has boards and projects — no manual creation needed.

## 5. Verify before running
```bash
# Check Jira version
curl -su admin:admin "http://<elb>/jira/rest/api/2/serverInfo" | python -m json.tool | grep version

# Check plugin is loaded
curl -su admin:admin "http://<elb>/jira/rest/skillsforjira/1/user" | python -m json.tool | head -5

# Check Worker Skillset field
curl -su admin:admin "http://<elb>/jira/rest/api/2/field" | python -c "import sys,json; print([f['name'] for f in json.load(sys.stdin) if 'Skillset' in f.get('name','')])"

# Check dashboard-diagnostics matches base URL
curl -su admin:admin -X POST "http://<elb>/jira/plugins/servlet/gadgets/dashboard-diagnostics" \
  --data "uri=http://<elb>/jira/secure/Dashboard.jspa" | head -1
# Must print: Dashboard Diagnostics: OK
```
