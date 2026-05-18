name: Content Contribution
description: Contribute new documentation, guides, or case studies
title: "[Content]: "
labels: ["content", "documentation", "triage"]
assignees:
  - ""
body:
  - type: markdown
    attributes:
      value: |
        Thanks for contributing to the Cloud FinOps Playbook!

  - type: input
    id: topic
    attributes:
      label: Topic Title
      description: Brief title for your contribution
      placeholder: ex. EKS Cost Optimization Guide
    validations:
      required: true

  - type: dropdown
    id: content_type
    attributes:
      label: Content Type
      description: What type of content are you contributing?
      options:
        - New Guide/Documentation
        - Case Study
        - Code Example/Script
        - Architecture Diagram
        - Checklist/Template
        - Update to Existing Content
    validations:
      required: true

  - type: dropdown
    id: section
    attributes:
      label: Target Section
      description: Where should this content be added?
      options:
        - foundations/
        - aws/
        - azure/
        - gcp/
        - kubernetes/
        - neoclouds/
        - ai-infrastructure/
        - openstack/
        - New Section (describe below)
    validations:
      required: true

  - type: textarea
    id: summary
    attributes:
      label: Summary
      description: Describe what you're contributing and why it's valuable
      placeholder: This guide covers...
    validations:
      required: true

  - type: textarea
    id: target_audience
    attributes:
      label: Target Audience
      description: Who is this content for? (e.g., platform engineers, SREs, ML engineers)
    validations:
      required: true

  - type: input
    id: cloud_provider
    attributes:
      label: Cloud Provider(s)
      description: Which cloud providers does this cover?
      placeholder: AWS, Azure, GCP, NeoClouds, Multi-cloud
    validations:
      required: false

  - type: textarea
    id: links
    attributes:
      label: Related Links/References
      description: Any relevant links, PRs, or references
    validations:
      required: false

  - type: checkboxes
    id: terms
    attributes:
      label: Code of Conduct
      description: By submitting this issue, you agree to follow our Code of Conduct
      options:
        - label: I agree to follow this project's Code of Conduct
          required: true
