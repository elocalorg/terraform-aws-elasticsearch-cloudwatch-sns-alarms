# terraform-aws-elasticsearch-cloudwatch-sns-alarms

Sets up CloudWatch alarms and SNS notifications for Elasticsearch.

**GitHub:** https://github.com/elocalorg/terraform-aws-elasticsearch-cloudwatch-sns-alarms
**Terraform Cloud registry:** app.terraform.io/elocal/elasticsearch-cloudwatch-sns-alarms/aws
**Latest version:** check the latest git tag

## Standards and patterns

Before making changes, read the relevant sub-pages under [Terraform](https://elocal.atlassian.net/wiki/spaces/GP/pages/2702475268/Terraform) in Confluence. Key topics covered there include module design principles, versioning rules, infrastructure patterns (INFRA stacks, single-image, multi-image), and the intermediary entities pattern.

## Confluence publish workflow

The README is auto-published to Confluence on each tag push using the centralized composite action. Do not copy the script locally:

```yaml
uses: elocalorg/github-workflows/.github/actions/publish-to-confluence@master
```

Org-level vars/secrets are pre-configured. The only repo-specific input is `parent_page_id` (`vars.CONFLUENCE_TERRAFORM_MODULES_PARENT_PAGE_ID`).

## Do not use the Confluence page for this module as a reference

This module's README will be auto-published to Confluence. That published page is a mirror of the README in this repo — consulting it as a source of truth creates a circular reference. Always use the README in this repo directly.
