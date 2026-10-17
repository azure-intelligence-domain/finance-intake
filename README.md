# finance-intake

The **finance domain's intake flow** ([ADR 0015](https://github.com/azure-intelligence-domain/ai-intelligence-domain/blob/main/docs/adr/0015-one-intake-per-domain.md)):
what the finance domain takes in from other domains. A Logic App (Consumption), owned by the
**finance-team**. It receives **OrderPlaced** and creates the sales order in SAP, through the gateway, with the
SAP System API; the System API then publishes SalesOrderAccepted / SalesOrderRefused and InvoiceIssued.
It also receives **OrderCancelled** (below).

| File | What |
|---|---|
| [`workflow.json`](workflow.json) | The flow: switch on the event type, map, call the System API |
| [`asyncapi.yaml`](asyncapi.yaml) | Consumer contract: what it receives (from the Order Process API's released contract) |
| [`infra/main.bicep`](infra/main.bicep) | The Logic App, its runtime identity, and its OAuth policy (the gateway only) |
| [`tests/`](tests/) | The flow against its contract |

```mermaid
sequenceDiagram
    autonumber
    participant EG as evgt-intplat-order
    participant G as apim-intplat-demo
    participant LA as finance-intake (Logic App)
    participant S as SAP System API
    EG->>G: OrderPlaced (subscription finance-intake, filtered on the event type)
    G->>LA: POST /intake/finance-intake → run (gateway identity, OAuth policy)
    LA->>G: POST /sap/sales-orders (runtime identity, Correlation-Id)
    G->>S: system-apis product: allow-listed caller
    S-->>LA: 201
    LA-->>G: 200
    Note over EG,S: The System API publishes SalesOrderAccepted / SalesOrderRefused and InvoiceIssued. A failure: 502, Event Grid retries, then dead-letters into the tap.
```

## Mapping

The flow maps the order domain's event onto the SAP System API's contract; nothing else.

```mermaid
flowchart LR
    ev["OrderPlaced<br/>(order domain contract)"] -- "workflow.json" --> req["/sap/sales-orders<br/>(SAP System API contract)"]
```

## When an order is cancelled (since 1.1.0)

**OrderCancelled** names what each domain has for the order ([ADR 0018](https://github.com/azure-intelligence-domain/ai-intelligence-domain/blob/main/docs/adr/0018-cancellation-reaches-every-domain.md)).
If it carries `erpOrderNumber`, the flow cancels the SAP sales order; a credit note reverses the invoice: `DELETE /sap/sales-orders/{erpOrderNumber}?reason=`, and the SAP System API
publishes **SalesOrderCancelled**. Without `erpOrderNumber` the SAP has nothing for this order yet, and the run ends
`200` without a call.

```mermaid
flowchart LR
    oc[["OrderCancelled<br/>erpOrderNumber"]] --> c{"erpOrderNumber set?"}
    c -- yes --> d["DELETE /sap/sales-orders/{erpOrderNumber}"] --> f[["SalesOrderCancelled"]]
    c -- no --> n["nothing to undo: 200"]
```

## When a delivery fails

The flow does not retry: if the System API refuses or fails, the run ends in `502 Delivery
Failed`, Event Grid retries the delivery (3 attempts), then dead-letters the event into the tap
storage (`sttapintplatdemo`, container `deadletters`, folder `finance-intake`). The Intelligence domain
sees it there; nobody has to search Logic App run histories.

## Cost

Pay per run, nothing polls: an idle flow costs nothing. A run is about five built-in actions plus
one HTTP call.

## Pipeline

`.github/workflows/deploy.yml`: contract check and flow tests, then on `main` (OIDC as
`id-intplat-finance-intake`): the Logic App, a smoke test (a call without the gateway's token gets `401`),
and a deployment event.
