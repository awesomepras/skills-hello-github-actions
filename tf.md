# ============================================================
# variables.tf
# ============================================================

variable "apigee_project_id" {
  type = string
}

variable "tenant_service_accounts" {
  type = map(object({
    project_id = string
    apm        = string
    env        = string
  }))
}

# ============================================================
# iam.tf
# ============================================================

resource "google_project_iam_member" "apigee_proxy_deployers" {

  for_each = var.tenant_service_accounts

  project = var.apigee_project_id

  role = "roles/apigee.apiAdmin"

  member = "serviceAccount:${each.value.env}-${each.value.apm}-actions@${each.value.project_id}.iam.gserviceaccount.com"
}

# ============================================================
# tenants.auto.tfvars
# ============================================================

apigee_project_id = "project-a"

tenant_service_accounts = {

  # Example existing tenant
  paym = {
    project_id = "payments-prod"
    apm        = "paym"
    env        = "pr"
  }

}

# ============================================================
# .github/ISSUE_TEMPLATE/apigee-access-request.yml
# ============================================================

name: Apigee Access Request
description: Request ApigeeX deployment access
title: "[APIGEE ACCESS] "
labels:
  - apigee-access

body:

  - type: input
    id: projectid
    attributes:
      label: ProjectID
      description: GCP Project ID
      placeholder: np-bfb8-sc-05u9uas
    validations:
      required: true

  - type: input
    id: apm
    attributes:
      label: APM
      description: 4 letter application code
      placeholder: bill
    validations:
      required: true

  - type: input
    id: serviceaccount
    attributes:
      label: Service Account
      description: Service account used by GitHub Actions
      placeholder: np-bfb8-actions@np-bfb8-sc-05u9uas.iam.gserviceaccount.com
    validations:
      required: true

  - type: dropdown
    id: env
    attributes:
      label: Environment
      options:
        - np
        - pr
    validations:
      required: true


# ============================================================
# .github/workflows/issue-create-tenant.yml
# ============================================================

name: Create Tenant From Issue

on:
  issues:
    types: [opened]

permissions:
  contents: write
  issues: read

jobs:

  create-tenant-entry:

    if: contains(github.event.issue.title, '[APIGEE ACCESS]')

    runs-on: ubuntu-latest

    steps:

      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Parse Issue Body
        id: parse
        run: |

          BODY="${{ github.event.issue.body }}"

          PROJECT_ID=$(echo "$BODY" | grep -i "ProjectID" | awk -F':' '{print $2}' | xargs)
          APM=$(echo "$BODY" | grep -i "APM" | awk -F':' '{print $2}' | xargs)
          SA=$(echo "$BODY" | grep -i "Service Account" | awk -F':' '{print $2}' | xargs)
          ENV=$(echo "$BODY" | grep -i "Environment" | awk -F':' '{print $2}' | xargs)

          echo "project_id=$PROJECT_ID" >> $GITHUB_OUTPUT
          echo "apm=$APM" >> $GITHUB_OUTPUT
          echo "sa=$SA" >> $GITHUB_OUTPUT
          echo "env=$ENV" >> $GITHUB_OUTPUT

      - name: Check Duplicate APM
        run: |

          if grep -q "${{ steps.parse.outputs.apm }} =" tenants.auto.tfvars; then
            echo "APM already exists!"
            exit 1
          fi

      - name: Create Branch
        run: |

          BRANCH="issue-${{ github.event.issue.number }}"
          git checkout -b $BRANCH

      - name: Update tenants.auto.tfvars
        run: |

          cat <<EOF >> tenants.auto.tfvars

  ${{ steps.parse.outputs.apm }} = {
    project_id = "${{ steps.parse.outputs.project_id }}"
    apm        = "${{ steps.parse.outputs.apm }}"
    env        = "${{ steps.parse.outputs.env }}"
  }

EOF

      - name: Commit Changes
        run: |

          git config user.name "github-actions"
          git config user.email "github-actions@github.com"

          git add tenants.auto.tfvars

          git commit -m "Add Apigee tenant ${{ steps.parse.outputs.apm }} from issue #${{ github.event.issue.number }}"

      - name: Push Branch
        run: |

          git push origin issue-${{ github.event.issue.number }}
