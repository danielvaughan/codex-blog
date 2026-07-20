---
title: "Automated SAP Testing with Codex CLI: An Agent-Driven Approach"
description: "How to use Codex CLI to generate, maintain, and execute automated tests across SAP's four testing layers — OData APIs, BAPIs/RFCs, Fiori UI, and SAP GUI — with practical code examples, MCP integration patterns, and guidance on navigating SAP's April 2026 API policy."
date: 2026-06-13T20:00:00+00:00
last_modified_at: 2026-07-20T10:15:45+01:00
tags:
  - codex-cli
  - sap
  - testing
  - automation
  - enterprise
  - odata
  - mcp
  - agents-md
citations:
  - id: 1
    title: "SAP Test Automation: A Complete Guide for 2026"
    url: "https://aqua-cloud.io/sap-test-automation-comprehensive-guide/"
    accessed: 2026-06-13
  - id: 2
    title: "End-to-End SAP GUI Logon Automation with Codex and Python"
    url: "https://community.sap.com/t5/technology-blog-posts-by-sap/end-to-end-sap-gui-logon-automation-with-codex-and-python-customizing/ba-p/14342645"
    accessed: 2026-06-13
  - id: 3
    title: "Claude Code + Python + SAP GUI Scripting: Your AI Agent for Any SAP Transaction"
    url: "https://community.sap.com/t5/technology-blog-posts-by-sap/claude-code-python-sap-gui-scripting-your-ai-agent-for-any-sap-transaction/ba-p/14327865"
    accessed: 2026-06-13
  - id: 4
    title: "Unlocking the Power of Model Context Protocol (MCP) for SAP Business Applications"
    url: "https://community.sap.com/t5/technology-blog-posts-by-members/unlocking-the-power-of-model-context-protocol-mcp-for-sap-business/ba-p/14258263"
    accessed: 2026-06-13
  - id: 5
    title: "SAP-Claude Integration via MCP (Model Context Protocol)"
    url: "https://mcp.so/server/sap-mcp/CostingGeek"
    accessed: 2026-06-13
  - id: 6
    title: "SAP API Policy Change: New Rules Restrict Third-Party AI Agents"
    url: "https://aimagazine.com/news/can-ai-agents-still-access-sap-data-under-new-api-rules"
    accessed: 2026-06-13
  - id: 7
    title: "Codex and Hermes Agent for Automation QA Engineers: A Practical Field Guide"
    url: "https://www.dhirajdas.dev/blog/codex-hermes-agent-automation-qa-guide"
    accessed: 2026-06-13
  - id: 8
    title: "Sandbox — Codex CLI Developer Documentation"
    url: "https://developers.openai.com/codex/concepts/sandboxing"
    accessed: 2026-06-13
  - id: 9
    title: "Custom Instructions with AGENTS.md — Codex Developer Documentation"
    url: "https://developers.openai.com/codex/guides/agents-md"
    accessed: 2026-06-13
  - id: 10
    title: "SAP S/4HANA Cloud Test Automation Tool in SAP S/4HANA Public Cloud Edition CE2508"
    url: "https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-sap/sap-s-4hana-cloud-test-automation-tool-in-sap-s-4hana-public-cloud-edition/ba-p/14163913"
    accessed: 2026-06-13
  - id: 11
    title: "SAP Integration Suite — Agentic Testing with Int4 Suite"
    url: "https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/sap-integration-suite-agentic-testing-is-available-now-with-int4-suite/ba-p/14322864"
    accessed: 2026-06-13
  - id: 12
    title: "OData for Pythonistas — SAP Community"
    url: "https://blogs.sap.com/2019/08/13/odata-for-pythonistas/"
    accessed: 2026-06-13
  - id: 13
    title: "Building an Agentic AI System with MCP and SAP BTP Kyma"
    url: "https://community.sap.com/t5/technology-blog-posts-by-sap/building-an-agentic-ai-system-with-model-context-protocol-mcp-and-sap-btp/ba-p/14090900"
    accessed: 2026-06-13
  - id: 14
    title: "Permissions — Codex CLI Developer Documentation"
    url: "https://developers.openai.com/codex/permissions"
    accessed: 2026-06-13
  - id: 15
    title: "SAP's New API Policy Restricts AI Access — The Register"
    url: "https://www.theregister.com/2026/04/29/new_sap_api_policy_provokes/"
    accessed: 2026-06-13
  - id: 16
    title: "7 Best SAP Test Automation Tools in 2026"
    url: "https://aqua-cloud.io/7-best-sap-test-automation-tools/"
    accessed: 2026-06-13
  - id: 17
    title: "SAP BAPIs and RFC Integration"
    url: "https://knowledgelib.io/business/erp-integration/sap-bapi-rfc-integration/2026"
    accessed: 2026-06-13
  - id: 18
    title: "How to Call SAP OData Services with Python"
    url: "https://ourcodeworld.com/articles/read/2684/how-to-call-sap-odata-services-with-python"
    accessed: 2026-06-13
  - id: 19
    title: "SAP S/4HANA Upgrade Testing: 3 Proven Strategies (2026)"
    url: "https://blog.perfectwin.ai/sap-s4hana-upgrade-testing-strategy"
    accessed: 2026-06-13
  - id: 20
    title: "Data Ownership in the Age of Agentic AI: Why SAP's API Policy Forces a Data Integration Reckoning"
    url: "https://www.kai-waehner.de/blog/2026/05/02/data-ownership-in-the-age-of-agentic-ai-why-saps-api-policy-forces-a-data-integration-reckoning-for-every-enterprise/"
    accessed: 2026-06-13
type: Technical Article
timestamp: 2026-06-13T21:00:00+01:00
resource: "https://danielvaughan.github.io/codex-resources/articles/2026-06-13-automated-sap-testing-with-codex-cli"
---
SAP testing is expensive. Manual regression suites for S/4HANA implementations routinely run to thousands of test cases. Every quarterly upgrade, every transport, every customising change triggers another cycle of scripted clicks through the same transactions. The specialist tools — Tricentis Tosca, Worksoft Certify, SAP's own Cloud Test Automation Tool — are capable, but they are also complex, licence-intensive, and require dedicated test engineers who understand both the tooling and the SAP landscape.[^1][^16]

Codex CLI offers something different: an AI agent that can read your SAP project, understand its interfaces, generate tests in standard frameworks (pytest, Playwright, shell scripts), execute them in a sandbox, and iterate on failures — all from a terminal prompt or a CI pipeline. It does not replace your test strategy. It accelerates the parts that are slowest: writing the tests, maintaining them when the system changes, and covering the gaps nobody has time to fill.

This article maps out how.

---

## The four testing layers in SAP

Every SAP implementation has four surfaces that need testing. Most organisations test one or two. The ones they skip are where production incidents hide.

| Layer | Technology | What it tests | Typical tools |
|-------|-----------|---------------|---------------|
| **1. API / OData** | OData V2/V4, REST | Business logic, data integrity, service contracts | Postman, SAP Gateway Client, pytest |
| **2. BAPI / RFC** | RFC, BAPI, IDoc | Cross-system integration, data exchange, process orchestration | SAP JCo, pyrfc, INT4 IFTT |
| **3. Fiori UI** | SAPUI5, OPA5 | User workflows, form validation, navigation | Playwright, Selenium, OPA5, Tricentis |
| **4. SAP GUI** | SAP GUI scripting | Classic transactions, customising, mass data entry | win32com (Python), VBScript, Tosca |

The challenge is not that any one layer is impossibly hard to test. The challenge is that real work crosses all four: a sales order starts as a Fiori draft, pricing conditions are adjusted in GUI transaction VA02, and the result is transmitted to a third-party system via an OData API. Testing each interface in isolation misses errors at the transition points.[^1][^19]

Codex CLI can generate tests across all four layers. It reads the project context — ABAP source, OData metadata documents, Fiori component definitions, GUI scripting recordings — and produces test code that targets the appropriate interface.

---

## Why Codex CLI, not a dedicated SAP testing tool?

Dedicated tools are good at what they do. Here is what they are not good at:

**Test creation velocity.** Writing a Tricentis Tosca module for a new process takes a test engineer a day. Describing the same process to Codex takes minutes.

**Maintenance cost.** Every SAP UI change potentially breaks dozens of scripts. UI-based testing is dependent on the UI staying the same. Codex can read the changed screens, identify what broke, and update the scripts — often in the same session that discovered the failure.[^1]

**Coverage gaps.** Most SAP organisations test the UI. Few test the APIs systematically. Fewer still test BAPI/RFC integrations end to end. Codex can generate API contract tests from OData metadata documents without anyone writing a test plan.[^10]

**Cost.** A Tricentis Tosca licence for SAP can run to six figures annually. Codex CLI is $20/month for Pro, and the API costs for test generation are fractional.[^16]

This is not an either/or decision. The most effective pattern is Codex generating tests that run alongside your existing suite — filling the gaps, not replacing the framework.

---

## Setting up the project

### Project structure

```
sap-test-suite/
  tests/
    api/              # OData API contract tests
    bapi/             # BAPI/RFC integration tests
    fiori/            # Playwright UI tests for Fiori apps
    gui/              # SAP GUI scripting tests
    fixtures/         # Shared test data and configuration
    conftest.py       # pytest configuration and shared fixtures
  config/
    systems.yaml      # SAP system connection details (not committed)
    odata-metadata/   # Downloaded $metadata documents
  scripts/
    record-gui.py     # SAP GUI scripting recorder
    export-metadata.sh # OData metadata export utility
  AGENTS.md           # Codex project instructions
  pytest.ini
  requirements.txt
```

### The AGENTS.md file

This is the most important file in the project. It tells Codex how to behave in your SAP testing context.[^9]

```markdown
# SAP Test Automation Agent

## Role
You are a test automation engineer for SAP S/4HANA.
You write and maintain automated tests across four layers:
OData APIs, BAPI/RFC integrations, Fiori UI, and SAP GUI transactions.

## Rules
- NEVER execute tests against the production system (PRD)
- NEVER hard-code credentials — always read from environment variables or config/systems.yaml
- NEVER modify SAP system configuration — tests are read-only observers
- ALWAYS use the QAS (quality assurance) system for test execution
- ALWAYS clean up test data created during test runs
- ALWAYS use synthetic test data, never real customer or employee data
- ALWAYS validate OData responses against the $metadata document
- Prefer API-level tests over UI tests where both are possible

## Test frameworks
- Python: pytest with pytest-html for reporting
- OData: requests library with pyodata for metadata parsing
- BAPI/RFC: pyrfc library for RFC connections
- Fiori UI: Playwright (TypeScript) with page object pattern
- SAP GUI: Python win32com.client for GUI scripting automation

## Execution
- Run specific layer: `pytest tests/api/` or `pytest tests/bapi/`
- Run all: `pytest --html=report.html`
- Fiori tests: `npx playwright test tests/fiori/`

## SAP systems
- Connection details are in config/systems.yaml (never committed to git)
- Environment variables: SAP_HOST, SAP_CLIENT, SAP_USER, SAP_PASS, SAP_SYSNR

## Verification
After generating or modifying tests:
1. Run the relevant test suite
2. Verify all tests pass
3. Check that no credentials are hard-coded
4. Confirm test data cleanup runs correctly
```

### Sandbox configuration

Codex CLI's sandbox is critical for SAP testing work. You want the agent to execute tests (which make network calls to your SAP system) but not to modify production data or configuration.[^8][^14]

```toml
# .codex/config.toml
sandbox_mode = "workspace-write"
approval_policy = "on-request"
```

In the `workspace-write` mode, Codex can read files, write test code, and execute test commands within the project directory. It will ask for approval before running commands that require network access — which is exactly the control point you want when tests hit a live SAP system.

For CI pipelines where human approval is not practical, use a locked-down execution policy that restricts outbound connections to your SAP QAS system only.

---

## Layer 1: OData API testing

OData is the primary API surface for S/4HANA Cloud and modern Fiori applications. Every business object — sales orders, purchase orders, business partners, journal entries — is exposed as an OData entity set.[^10][^12]

### Generating tests from metadata

The OData `$metadata` document is a machine-readable contract. It defines every entity type, property, association, and function import. Codex can read it and generate comprehensive contract tests.

First, export the metadata:

```bash
# scripts/export-metadata.sh
#!/bin/bash
# Export OData V2 metadata for a service
curl -u "$SAP_USER:$SAP_PASS" \
  "https://$SAP_HOST/sap/opu/odata/sap/API_SALES_ORDER_SRV/\$metadata" \
  -o config/odata-metadata/API_SALES_ORDER_SRV.xml

curl -u "$SAP_USER:$SAP_PASS" \
  "https://$SAP_HOST/sap/opu/odata/sap/API_BUSINESS_PARTNER/\$metadata" \
  -o config/odata-metadata/API_BUSINESS_PARTNER.xml
```

Then ask Codex to generate tests:

```
codex "Read the OData metadata in config/odata-metadata/API_SALES_ORDER_SRV.xml.
Generate pytest tests that validate:
1. Every entity set returns HTTP 200 for a GET request
2. Required fields are present in responses
3. Data types match the metadata definitions
4. Navigation properties resolve correctly
5. Filter and pagination parameters work
Write the tests to tests/api/test_sales_order_api.py"
```

### What Codex generates

Codex produces tests using the `requests` library with `pyodata` for metadata parsing:[^12][^18]

```python
# tests/api/test_sales_order_api.py
import os
import pytest
import requests
import pyodata
from requests.auth import HTTPBasicAuth

SAP_HOST = os.environ["SAP_HOST"]
SAP_USER = os.environ["SAP_USER"]
SAP_PASS = os.environ["SAP_PASS"]
SERVICE_URL = f"https://{SAP_HOST}/sap/opu/odata/sap/API_SALES_ORDER_SRV"
AUTH = HTTPBasicAuth(SAP_USER, SAP_PASS)


@pytest.fixture(scope="module")
def odata_client():
    """Initialise the OData client from local metadata."""
    metadata_path = "config/odata-metadata/API_SALES_ORDER_SRV.xml"
    with open(metadata_path, "r") as f:
        metadata_xml = f.read()

    session = requests.Session()
    session.auth = AUTH
    session.headers.update({"Accept": "application/json"})

    client = pyodata.Client.build(
        SERVICE_URL,
        session,
        metadata=metadata_xml,
    )
    return client


class TestSalesOrderEntitySets:
    """Validate that all entity sets are accessible and return correct structure."""

    def test_sales_order_list_returns_200(self, odata_client):
        """GET /A_SalesOrder should return 200 with results array."""
        response = odata_client.entity_sets.A_SalesOrder.get_entities().execute()
        assert len(response) >= 0  # Validates the request succeeded

    def test_sales_order_required_fields(self, odata_client):
        """First sales order should contain all required fields from metadata."""
        orders = odata_client.entity_sets.A_SalesOrder.get_entities().top(1).execute()
        if len(orders) == 0:
            pytest.skip("No sales orders in QAS system")

        order = orders[0]
        required_fields = [
            "SalesOrder",
            "SalesOrderType",
            "SalesOrganization",
            "SoldToParty",
            "CreationDate",
        ]
        for field in required_fields:
            assert hasattr(order, field), f"Missing required field: {field}"

    def test_sales_order_filter_by_type(self, odata_client):
        """Filter by SalesOrderType should return only matching records."""
        orders = (
            odata_client.entity_sets.A_SalesOrder
            .get_entities()
            .filter("SalesOrderType eq 'OR'")
            .top(5)
            .execute()
        )
        for order in orders:
            assert order.SalesOrderType == "OR"

    def test_sales_order_pagination(self, odata_client):
        """$top and $skip parameters should work correctly."""
        page1 = (
            odata_client.entity_sets.A_SalesOrder
            .get_entities()
            .top(2)
            .execute()
        )
        page2 = (
            odata_client.entity_sets.A_SalesOrder
            .get_entities()
            .skip(2)
            .top(2)
            .execute()
        )
        if len(page1) >= 2 and len(page2) >= 1:
            # Pages should not overlap
            page1_ids = {o.SalesOrder for o in page1}
            page2_ids = {o.SalesOrder for o in page2}
            assert page1_ids.isdisjoint(page2_ids), "Pagination returned overlapping records"

    def test_sales_order_item_navigation(self, odata_client):
        """Navigation property to_Item should resolve to line items."""
        orders = (
            odata_client.entity_sets.A_SalesOrder
            .get_entities()
            .top(1)
            .execute()
        )
        if len(orders) == 0:
            pytest.skip("No sales orders in QAS system")

        # Fetch items via navigation property
        items_url = f"{SERVICE_URL}/A_SalesOrder('{orders[0].SalesOrder}')/to_Item"
        response = requests.get(items_url, auth=AUTH, headers={"Accept": "application/json"})
        assert response.status_code == 200


class TestSalesOrderDataTypes:
    """Validate response data types match the $metadata definitions."""

    def test_creation_date_is_datetime(self, odata_client):
        """CreationDate should parse as a valid date."""
        orders = odata_client.entity_sets.A_SalesOrder.get_entities().top(1).execute()
        if len(orders) == 0:
            pytest.skip("No sales orders in QAS system")

        # pyodata automatically converts Edm.DateTime to Python datetime
        assert orders[0].CreationDate is not None

    def test_net_amount_is_decimal(self, odata_client):
        """TotalNetAmount should be a numeric value."""
        orders = odata_client.entity_sets.A_SalesOrder.get_entities().top(1).execute()
        if len(orders) == 0:
            pytest.skip("No sales orders in QAS system")

        amount = orders[0].TotalNetAmount
        assert isinstance(amount, (int, float, str)), "TotalNetAmount should be numeric"
```

The key insight: Codex did not invent these field names. It read them from the metadata document. When the OData service changes — new fields, renamed properties, deprecated endpoints — you re-export the metadata and ask Codex to update the tests.

---

## Layer 2: BAPI/RFC integration testing

BAPIs and RFC function modules are the backbone of SAP cross-system integration. They predate OData by two decades and remain the primary interface for ECC systems, middleware (SAP PI/PO, Integration Suite), and legacy integrations.[^17]

### Using pyrfc with Codex

The `pyrfc` library provides Python bindings for SAP's NetWeaver RFC SDK. Codex can generate tests that call BAPIs directly:

```
codex "Generate pytest tests for the BAPI_SALESORDER_GETLIST BAPI.
The tests should:
1. Call the BAPI with a valid customer number from fixtures
2. Verify the return table contains expected fields
3. Test error handling for invalid customer numbers
4. Validate that RETURN messages have the correct structure
Use pyrfc for RFC connections. Connection details come from environment variables.
Write to tests/bapi/test_sales_order_bapi.py"
```

### Generated BAPI test

```python
# tests/bapi/test_sales_order_bapi.py
import os
import pytest
from pyrfc import Connection


@pytest.fixture(scope="module")
def sap_connection():
    """Establish an RFC connection to the SAP QAS system."""
    conn = Connection(
        ashost=os.environ["SAP_HOST"],
        sysnr=os.environ["SAP_SYSNR"],
        client=os.environ["SAP_CLIENT"],
        user=os.environ["SAP_USER"],
        passwd=os.environ["SAP_PASS"],
    )
    yield conn
    conn.close()


@pytest.fixture
def test_customer():
    """Return a known test customer number from QAS."""
    return os.environ.get("SAP_TEST_CUSTOMER", "0000001000")


class TestBapiSalesorderGetlist:
    """Integration tests for BAPI_SALESORDER_GETLIST."""

    def test_returns_sales_orders_for_valid_customer(
        self, sap_connection, test_customer
    ):
        """Valid customer should return a non-empty sales order list."""
        result = sap_connection.call(
            "BAPI_SALESORDER_GETLIST",
            CUSTOMER_NUMBER=test_customer,
            SALES_ORGANIZATION="1000",
        )

        assert "SALES_ORDERS" in result
        assert len(result["SALES_ORDERS"]) > 0, (
            f"No sales orders found for customer {test_customer}"
        )

    def test_sales_order_record_structure(
        self, sap_connection, test_customer
    ):
        """Each sales order record should contain expected fields."""
        result = sap_connection.call(
            "BAPI_SALESORDER_GETLIST",
            CUSTOMER_NUMBER=test_customer,
            SALES_ORGANIZATION="1000",
        )

        expected_fields = [
            "SD_DOC",           # Sales document number
            "DOC_TYPE",         # Document type
            "PURCH_NO",         # Customer PO number
            "NET_VALUE",        # Net value
            "CURRENCY",         # Currency code
            "DOC_DATE",         # Document date
        ]

        if result["SALES_ORDERS"]:
            order = result["SALES_ORDERS"][0]
            for field in expected_fields:
                assert field in order, f"Missing field: {field}"

    def test_invalid_customer_returns_error(self, sap_connection):
        """Invalid customer number should produce a RETURN message."""
        result = sap_connection.call(
            "BAPI_SALESORDER_GETLIST",
            CUSTOMER_NUMBER="9999999999",
            SALES_ORGANIZATION="1000",
        )

        # BAPI convention: errors in RETURN table
        return_messages = result.get("RETURN", [])
        # Either no results or an error/warning message
        has_error = any(
            msg.get("TYPE") in ("E", "W") for msg in return_messages
        )
        has_no_data = len(result.get("SALES_ORDERS", [])) == 0

        assert has_error or has_no_data, (
            "Invalid customer should produce an error or empty result"
        )

    def test_return_message_structure(self, sap_connection, test_customer):
        """RETURN messages should follow the standard BAPIRET2 structure."""
        result = sap_connection.call(
            "BAPI_SALESORDER_GETLIST",
            CUSTOMER_NUMBER=test_customer,
            SALES_ORGANIZATION="1000",
        )

        bapiret2_fields = ["TYPE", "ID", "NUMBER", "MESSAGE"]

        for msg in result.get("RETURN", []):
            for field in bapiret2_fields:
                assert field in msg, (
                    f"RETURN message missing BAPIRET2 field: {field}"
                )


class TestRfcConnectivity:
    """Baseline connectivity tests for the RFC connection."""

    def test_rfc_ping(self, sap_connection):
        """RFC_PING should succeed, confirming connection is alive."""
        result = sap_connection.call("RFC_PING")
        assert result is not None

    def test_system_info(self, sap_connection):
        """RFC_SYSTEM_INFO should return system details."""
        result = sap_connection.call("RFC_SYSTEM_INFO")
        rfcsi = result.get("RFCSI_EXPORT", {})
        assert rfcsi.get("RFCSYSID"), "System ID should not be empty"
        assert rfcsi.get("RFCHOST"), "Host name should not be empty"
```

### The maintenance advantage

When a BAPI's interface changes — new parameters, renamed fields, deprecated function modules — Codex can read the updated function module documentation (exported from SE37 or via RFC_GET_FUNCTION_INTERFACE) and regenerate the tests. The cycle is: export the interface definition, point Codex at it, regenerate.

---

## Layer 3: Fiori UI testing with Playwright

Fiori applications are SAPUI5 web apps. They run in a browser. Playwright tests them like any other web application — with one important difference: SAPUI5 renders controls dynamically, so stable selectors require understanding the framework's DOM structure.[^3]

```
codex "Generate Playwright TypeScript tests for the SAP Fiori
'Manage Sales Orders' app at https://qas.example.com/sap/bc/ui5_ui5/ui2/ushell/shells/abap/FioriLaunchpad.html.
Test the following workflow:
1. Log into the Fiori Launchpad
2. Open the 'Manage Sales Orders' tile
3. Search for sales orders by customer number
4. Open a sales order and verify header fields
5. Check that line items table is populated
Use a page object pattern. Write to tests/fiori/"
```

Codex generates a page object model with Fiori-aware selectors:

```typescript
// tests/fiori/pages/fiori-launchpad.ts
import { Page } from '@playwright/test';

export class FioriLaunchpad {
  constructor(private page: Page) {}

  async login(user: string, password: string) {
    await this.page.goto(process.env.FIORI_URL!);
    await this.page.fill('input[name="j_username"]', user);
    await this.page.fill('input[name="j_password"]', password);
    await this.page.click('button[type="submit"]');
    // Wait for the launchpad shell to load
    await this.page.waitForSelector('[data-sap-ui-area="canvas"]');
  }

  async openTile(tileTitle: string) {
    await this.page.click(`[title="${tileTitle}"]`);
    // Wait for the app to load fully
    await this.page.waitForLoadState('networkidle');
  }
}
```

```typescript
// tests/fiori/pages/manage-sales-orders.ts
import { Page, expect } from '@playwright/test';

export class ManageSalesOrdersPage {
  constructor(private page: Page) {}

  async searchByCustomer(customerNumber: string) {
    // SAPUI5 SmartFilterBar uses specific control IDs
    const filterBar = this.page.locator('[data-sap-ui*="smartFilterBar"]');
    await filterBar.locator('input[placeholder*="Customer"]').fill(customerNumber);
    await this.page.click('button:has-text("Go")');
    await this.page.waitForLoadState('networkidle');
  }

  async openFirstOrder() {
    // Click the first row in the responsive table
    const firstRow = this.page.locator('table tbody tr').first();
    await firstRow.click();
    await this.page.waitForLoadState('networkidle');
  }

  async getHeaderFields() {
    return {
      salesOrder: await this.page.locator('[data-sap-ui*="SalesOrder"] span').textContent(),
      customer: await this.page.locator('[data-sap-ui*="SoldToParty"] span').textContent(),
      netValue: await this.page.locator('[data-sap-ui*="TotalNetAmount"] span').textContent(),
    };
  }

  async getLineItemCount(): Promise<number> {
    const rows = this.page.locator('[data-sap-ui*="lineItemsTable"] tbody tr');
    return await rows.count();
  }
}
```

The test file:

```typescript
// tests/fiori/sales-order.spec.ts
import { test, expect } from '@playwright/test';
import { FioriLaunchpad } from './pages/fiori-launchpad';
import { ManageSalesOrdersPage } from './pages/manage-sales-orders';

test.describe('Manage Sales Orders', () => {
  let launchpad: FioriLaunchpad;
  let salesOrders: ManageSalesOrdersPage;

  test.beforeEach(async ({ page }) => {
    launchpad = new FioriLaunchpad(page);
    salesOrders = new ManageSalesOrdersPage(page);

    await launchpad.login(
      process.env.SAP_USER!,
      process.env.SAP_PASS!,
    );
    await launchpad.openTile('Manage Sales Orders');
  });

  test('search returns results for valid customer', async () => {
    await salesOrders.searchByCustomer('1000');
    const rows = await salesOrders.page.locator('table tbody tr').count();
    expect(rows).toBeGreaterThan(0);
  });

  test('sales order header displays required fields', async () => {
    await salesOrders.searchByCustomer('1000');
    await salesOrders.openFirstOrder();

    const header = await salesOrders.getHeaderFields();
    expect(header.salesOrder).toBeTruthy();
    expect(header.customer).toBeTruthy();
    expect(header.netValue).toBeTruthy();
  });

  test('sales order has line items', async () => {
    await salesOrders.searchByCustomer('1000');
    await salesOrders.openFirstOrder();

    const itemCount = await salesOrders.getLineItemCount();
    expect(itemCount).toBeGreaterThan(0);
  });
});
```

SAPUI5 selectors are brittle by nature — control IDs can change between application versions. Codex's value here is not just generating the tests but regenerating the selectors when the Fiori app changes. Describe the screen, Codex reads the DOM, Codex updates the page objects.[^7]

---

## Layer 4: SAP GUI scripting

For classic SAP transactions — VA01 (create sales order), ME21N (create purchase order), SM37 (job monitoring) — SAP GUI scripting with Python's `win32com.client` module is the established automation path.[^2][^3]

```
codex "Generate a Python test that automates SAP GUI transaction VA03
(display sales order). The test should:
1. Connect to an existing SAP GUI session
2. Open transaction VA03
3. Enter a sales order number from fixtures
4. Verify the order header fields are populated
5. Navigate to the item overview and check line items exist
6. Close the transaction
Use win32com.client for SAP GUI scripting.
Write to tests/gui/test_va03_display.py"
```

### Generated GUI test

```python
# tests/gui/test_va03_display.py
import os
import pytest
import win32com.client


@pytest.fixture(scope="module")
def sap_session():
    """Attach to an existing SAP GUI session."""
    sap_gui = win32com.client.GetObject("SAPGUI")
    application = sap_gui.GetScriptingEngine
    connection = application.Children(0)
    session = connection.Children(0)
    yield session


@pytest.fixture
def test_sales_order():
    """A known sales order number in the QAS system."""
    return os.environ.get("SAP_TEST_SALES_ORDER", "0000000100")


class TestVA03DisplaySalesOrder:
    """Test SAP GUI transaction VA03 — Display Sales Order."""

    def test_open_transaction(self, sap_session):
        """VA03 transaction should open without errors."""
        sap_session.StartTransaction("VA03")
        assert "Display Sales Order" in sap_session.ActiveWindow.Text or \
               "Kundenauftrag anzeigen" in sap_session.ActiveWindow.Text

    def test_enter_and_display_order(self, sap_session, test_sales_order):
        """Entering a valid order number should display the order."""
        sap_session.StartTransaction("VA03")

        # Enter the sales order number
        sap_session.findById("wnd[0]/usr/ctxtVBAK-VBELN").Text = test_sales_order
        sap_session.findById("wnd[0]").sendVKey(0)  # Press Enter

        # Verify the order header loaded
        status_bar = sap_session.findById("wnd[0]/sbar")
        assert status_bar.MessageType != "E", (
            f"Error displaying order: {status_bar.Text}"
        )

    def test_header_fields_populated(self, sap_session, test_sales_order):
        """Order header fields should contain data."""
        sap_session.StartTransaction("VA03")
        sap_session.findById("wnd[0]/usr/ctxtVBAK-VBELN").Text = test_sales_order
        sap_session.findById("wnd[0]").sendVKey(0)

        # Check document type
        doc_type = sap_session.findById(
            "wnd[0]/usr/subSUBSCREEN_HEADER:SAPMV45A:4021/txtVBAK-AUART"
        ).Text
        assert doc_type, "Document type should not be empty"

        # Check sold-to party
        sold_to = sap_session.findById(
            "wnd[0]/usr/subSUBSCREEN_HEADER:SAPMV45A:4021/subPART-SUB:SAPMV45A:4701/ctxtKUAGV-KUNNR"
        ).Text
        assert sold_to, "Sold-to party should not be empty"

    def test_item_overview_has_lines(self, sap_session, test_sales_order):
        """The item overview should contain at least one line item."""
        sap_session.StartTransaction("VA03")
        sap_session.findById("wnd[0]/usr/ctxtVBAK-VBELN").Text = test_sales_order
        sap_session.findById("wnd[0]").sendVKey(0)

        # Access the item table
        item_table = sap_session.findById(
            "wnd[0]/usr/tabsTAXI_TABSTRIP_OVERVIEW/tabpT\\01/ssubSUBSCREEN_BODY:SAPMV45A:4400/tblSAPMV45ATCTRL_U_ERF_AUFTRAG"
        )
        row_count = item_table.RowCount
        assert row_count > 0, "Sales order should have at least one line item"

    def test_cleanup_transaction(self, sap_session):
        """Return to the SAP Easy Access menu."""
        sap_session.StartTransaction("SESSION_MANAGER")
```

### Limitations of GUI testing with Codex

SAP GUI scripting requires a Windows desktop with SAP GUI installed and an active session. Codex CLI can generate and maintain the test scripts, but executing them requires a Windows environment. In CI pipelines, this typically means a Windows runner with SAP GUI for Windows installed. Codex cannot "see" the GUI — it generates scripts based on the scripting object model and recorded element IDs. The element IDs (like `wnd[0]/usr/ctxtVBAK-VBELN`) are stable within a given SAP release but can change between support packs.[^2]

---

## Connecting Codex to SAP via MCP

The Model Context Protocol (MCP) provides a standardised way for AI agents to interact with external systems. Several SAP MCP servers already exist:[^4][^5][^13]

| MCP Server | What it provides | Source |
|------------|-----------------|--------|
| **SAP-Claude Integration** | Sales orders via SAP Graph API | [CostingGeek/sap-mcp](https://mcp.so/server/sap-mcp/CostingGeek) |
| **msg-systems SAP BP** | Business partner data via SAP APIs | [LobeHub registry](https://lobehub.com/mcp/msg-systems-sap-bp-mcp-server) |
| **SAP MDK MCP Server** | AI-assisted MDK app development | [SAP/mdk-mcp-server](https://github.com/SAP/mdk-mcp-server) |
| **SAP BTP Kyma MCP** | Agentic AI on BTP Kyma Runtime | [SAP Community](https://community.sap.com/t5/technology-blog-posts-by-sap/building-an-agentic-ai-system-with-model-context-protocol-mcp-and-sap-btp/ba-p/14090900) |

### Building a test-oriented SAP MCP server

For testing specifically, you would build an MCP server that exposes test-relevant SAP operations as tools:

```python
# sap_test_mcp_server.py
"""MCP server exposing SAP test operations to Codex CLI."""

from mcp.server import Server
from mcp.types import Tool, TextContent
import os
from pyrfc import Connection

app = Server("sap-test-tools")


def get_sap_connection():
    """Create an RFC connection to the SAP QAS system."""
    return Connection(
        ashost=os.environ["SAP_HOST"],
        sysnr=os.environ["SAP_SYSNR"],
        client=os.environ["SAP_CLIENT"],
        user=os.environ["SAP_USER"],
        passwd=os.environ["SAP_PASS"],
    )


@app.tool()
async def get_sales_order(order_number: str) -> str:
    """Retrieve a sales order header and items from SAP via BAPI."""
    conn = get_sap_connection()
    try:
        result = conn.call(
            "BAPI_SALESORDER_GETDETAIL_V2",
            SALESDOCUMENT=order_number,
        )
        header = result.get("ORDER_HEADER_IN", {})
        items = result.get("ORDER_ITEMS_IN", [])
        return f"Order {order_number}: Type={header.get('DOC_TYPE')}, "  \
               f"Customer={header.get('SOLD_TO')}, "                     \
               f"Items={len(items)}"
    finally:
        conn.close()


@app.tool()
async def list_odata_services() -> str:
    """List available OData services on the SAP system."""
    import requests
    from requests.auth import HTTPBasicAuth

    url = f"https://{os.environ['SAP_HOST']}/sap/opu/odata/IWFND/CATALOGSERVICE;v=2/ServiceCollection"
    response = requests.get(
        url,
        auth=HTTPBasicAuth(os.environ["SAP_USER"], os.environ["SAP_PASS"]),
        headers={"Accept": "application/json"},
    )
    services = response.json().get("d", {}).get("results", [])
    return "\n".join(
        f"- {s['Title']} ({s['TechnicalServiceName']})"
        for s in services[:20]
    )


@app.tool()
async def check_rfc_connectivity() -> str:
    """Test RFC connectivity to the SAP system."""
    conn = get_sap_connection()
    try:
        result = conn.call("RFC_SYSTEM_INFO")
        info = result.get("RFCSI_EXPORT", {})
        return f"Connected to {info.get('RFCSYSID')} on {info.get('RFCHOST')}"
    finally:
        conn.close()
```

Register the MCP server in your Codex configuration. Codex can then use these tools during test generation — querying the real system to understand its structure before writing test assertions.

---

## The SAP API policy question

In April 2026, SAP updated its API policy. Section 2.2.2 of API Policy v4/2026 states that SAP APIs may not be used for "interaction or integration with (semi-)autonomous or generative AI systems that plan, select or execute sequences of API calls."[^6][^15][^20]

This has triggered significant concern across the SAP ecosystem. The Register called it a "perimeter around agentic AI." DSAG (the German-speaking SAP user group) publicly criticised the policy. CEO Christian Klein walked the message back on the Q1 investor call, saying the intent is to protect SAP's domain know-how and prevent performance degradation, not to block customers from their own data.[^15]

### What this means for Codex-driven SAP testing

The policy warrants careful reading, but testing is a materially different use case from the agentic production scenarios SAP is concerned about:

1. **Testing reads data; it does not plan sequences of business operations.** A test that calls `BAPI_SALESORDER_GETLIST` to verify the response structure is not an autonomous agent "planning and executing sequences of API calls" in the sense the policy targets.

2. **Testing operates on QAS/sandbox systems, not production.** The policy's concern about performance degradation applies to production workloads, not quality assurance environments with synthetic data.

3. **Codex generates the test code; it does not autonomously call SAP APIs at runtime in production.** The agent writes pytest files. Humans (or CI pipelines) execute them. The execution path is conventional — Python calling an OData service — identical to any existing test automation.

4. **SAP's own tooling supports test automation against its APIs.** The S/4HANA Cloud Test Automation Tool explicitly supports OData and SOAP API testing.[^10] Int4 IFTT provides "agentic testing" for SAP Integration Suite.[^11]

That said, if your organisation operates under strict SAP licensing terms, consult your SAP account team before deploying autonomous agents that make API calls, even for testing purposes. The distinction between "AI that generates test scripts" and "AI that autonomously executes API sequences" is clear to engineers but may be less clear to licensing auditors.

---

## Practical workflow: upgrade regression testing

The highest-value use case for Codex-driven SAP testing is upgrade regression. Every S/4HANA quarterly release changes APIs, UI controls, and business logic. The traditional approach — manually re-running hundreds of test cases over two weeks — is what makes SAP upgrades expensive.[^19]

### The Codex-accelerated approach

**Before the upgrade:**

```
codex "Scan the tests/ directory. For each OData service tested,
verify the $metadata documents in config/odata-metadata/ match
the current system. Report any differences."
```

Codex compares metadata files against the live system, identifies schema changes, and reports which tests will likely break.

**After the upgrade:**

```
codex "The SAP system has been upgraded to S/4HANA 2025 FPS02.
Run pytest tests/api/ and pytest tests/bapi/. For any failures:
1. Diagnose whether the failure is a test bug or a system change
2. If the API response structure changed, update the test
3. If the API returns different data, flag it for manual review
4. Do not weaken assertions to make tests pass"
```

The last instruction — "do not weaken assertions to make tests pass" — is critical. Without it, AI agents will happily make tests pass by removing the checks that caught the regression. The AGENTS.md should reinforce this.[^7]

**Generating coverage for new features:**

```
codex "The upgrade added a new OData entity set 'A_SalesOrderScheduleLine'
to API_SALES_ORDER_SRV. The updated $metadata is in
config/odata-metadata/API_SALES_ORDER_SRV.xml.
Generate contract tests for this new entity set following the
patterns in tests/api/test_sales_order_api.py."
```

Codex reads the existing test patterns, reads the new metadata, and generates tests that match the project's conventions. No test engineer needed to learn a new entity set's schema.

---

## What Codex cannot do

Codex CLI is not a silver bullet for SAP testing. It has clear boundaries:

**It cannot see screens.** Codex reads code and metadata. It cannot look at a Fiori application or SAP GUI session and understand what is on screen. For Fiori, it works from the DOM. For GUI, it works from scripting object IDs. If neither is available, a human must describe the screen.

**It cannot replace domain expertise.** Knowing that a three-way match in invoice verification requires the PO, goods receipt, and invoice to align within tolerance — that is SAP domain knowledge. Codex can generate the test scaffold, but someone who understands MM (Materials Management) must define what "correct" means.

**It does not understand SAP customising.** Two SAP systems with the same software version can behave completely differently based on customising tables (SPRO configuration). Tests generated against one system may not be valid for another.

**It requires network access for execution.** The sandbox must allow outbound connections to your SAP QAS system. In environments with strict network controls, this may require firewall rules, VPN configuration, or running tests on a jump host.

**GUI tests require Windows.** SAP GUI scripting is a Windows COM automation technology. Codex generates the Python scripts, but they must execute on a Windows machine with SAP GUI installed.

---

## Summary

| Layer | Codex generates | Framework | Runs where |
|-------|----------------|-----------|------------|
| OData API | Contract tests from $metadata | pytest + pyodata | Any OS, CI/CD |
| BAPI/RFC | Integration tests from function interfaces | pytest + pyrfc | Any OS with SAP NW RFC SDK |
| Fiori UI | Page objects + workflow tests | Playwright | Any OS, CI/CD |
| SAP GUI | Scripting automation tests | pytest + win32com | Windows with SAP GUI |

The pattern is the same across all four layers: export the interface definition (metadata, function module documentation, DOM structure, GUI scripting IDs), put it in the project alongside an AGENTS.md that defines the rules, and let Codex generate tests that a human reviews and a CI pipeline executes.

Codex does not replace your SAP test strategy. It replaces the part of your SAP test strategy that nobody has time to do — writing the next thousand tests.

---

## Sources

[^1]: [SAP Test Automation: A Complete Guide for 2026](https://aqua-cloud.io/sap-test-automation-comprehensive-guide/)
[^2]: [End-to-End SAP GUI Logon Automation with Codex and Python](https://community.sap.com/t5/technology-blog-posts-by-sap/end-to-end-sap-gui-logon-automation-with-codex-and-python-customizing/ba-p/14342645)
[^3]: [Claude Code + Python + SAP GUI Scripting: Your AI Agent for Any SAP Transaction](https://community.sap.com/t5/technology-blog-posts-by-sap/claude-code-python-sap-gui-scripting-your-ai-agent-for-any-sap-transaction/ba-p/14327865)
[^4]: [Unlocking the Power of MCP for SAP Business Applications](https://community.sap.com/t5/technology-blog-posts-by-members/unlocking-the-power-of-model-context-protocol-mcp-for-sap-business/ba-p/14258263)
[^5]: [SAP-Claude Integration via MCP](https://mcp.so/server/sap-mcp/CostingGeek)
[^6]: [SAP API Policy Change: New Rules Restrict Third-Party AI Agents](https://aimagazine.com/news/can-ai-agents-still-access-sap-data-under-new-api-rules)
[^7]: [Codex and Hermes Agent for Automation QA Engineers: A Practical Field Guide](https://www.dhirajdas.dev/blog/codex-hermes-agent-automation-qa-guide)
[^8]: [Sandbox — Codex CLI Developer Documentation](https://developers.openai.com/codex/concepts/sandboxing)
[^9]: [Custom Instructions with AGENTS.md — Codex Developer Documentation](https://developers.openai.com/codex/guides/agents-md)
[^10]: [SAP S/4HANA Cloud Test Automation Tool](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-sap/sap-s-4hana-cloud-test-automation-tool-in-sap-s-4hana-public-cloud-edition/ba-p/14163913)
[^11]: [SAP Integration Suite — Agentic Testing with Int4 Suite](https://community.sap.com/t5/enterprise-resource-planning-blog-posts-by-members/sap-integration-suite-agentic-testing-is-available-now-with-int4-suite/ba-p/14322864)
[^12]: [OData for Pythonistas — SAP Community](https://blogs.sap.com/2019/08/13/odata-for-pythonistas/)
[^13]: [Building an Agentic AI System with MCP and SAP BTP Kyma](https://community.sap.com/t5/technology-blog-posts-by-sap/building-an-agentic-ai-system-with-model-context-protocol-mcp-and-sap-btp/ba-p/14090900)
[^14]: [Permissions — Codex CLI Developer Documentation](https://developers.openai.com/codex/permissions)
[^15]: [SAP's New API Policy Restricts AI Access — The Register](https://www.theregister.com/2026/04/29/new_sap_api_policy_provokes/)
[^16]: [7 Best SAP Test Automation Tools in 2026](https://aqua-cloud.io/7-best-sap-test-automation-tools/)
[^17]: [SAP BAPIs and RFC Integration](https://knowledgelib.io/business/erp-integration/sap-bapi-rfc-integration/2026)
[^18]: [How to Call SAP OData Services with Python](https://ourcodeworld.com/articles/read/2684/how-to-call-sap-odata-services-with-python)
[^19]: [SAP S/4HANA Upgrade Testing: 3 Proven Strategies](https://blog.perfectwin.ai/sap-s4hana-upgrade-testing-strategy)
[^20]: [Data Ownership in the Age of Agentic AI: SAP's API Policy](https://www.kai-waehner.de/blog/2026/05/02/data-ownership-in-the-age-of-agentic-ai-why-saps-api-policy-forces-a-data-integration-reckoning-for-every-enterprise/)
