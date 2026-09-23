# STP Purchasing Spend Overview — source checklist and generation prompt

Updated September 23, 2026.

## Scope and assumptions

Proceed on the user's instruction to assume the identified CDS views are available. Real dimension lists may come from CDS views or transparent tables. Transaction structures and field names should follow the user's supplied samples. The fields below describe required business information, not a claim that every field is exposed by every installed CDS view.

The deliverable is the first purchasing overview page: invoiced spend, comparable-period change, non-managed spend, off-contract PO value, overdue open PO value, monthly trend, category and supplier rankings, contract use, potential leakage, unused contracts and invoice detail. Supplier evaluation, PR approval/cycle time, no-touch rate, invoice price variance, detailed contract consumption and IM metrics are outside this generation request.

These are proposed POC calculations. Exact numerical parity with individual Fiori apps is not an acceptance criterion until their populations and definitions have been reconciled.

## Metric-to-source checklist

| Displayed measure or visual | Sources and required information | Calculation and date basis |
|---|---|---|
| Invoiced purchasing spend; monthly trend; year-over-year change | C_SupplierInvoiceItemDEX; C_SupplierInvoiceDEX; non-PO supplement where needed. Complete invoice/year/item keys, invoicing supplier, company code, posting date, signed net amount, currency, posting/reversal status. | Sum posted, in-scope invoice-item net amounts, including signed credits/reversals, excluding tax. Group by posting year/month. Percentage change = (current − comparable prior)/comparable prior; N/A for zero denominator. No cash-payment interpretation. |
| Supplier/category rankings and invoice detail | Above invoice population; I_Supplier; C_PurchaseOrderItemDEX for PO category context; material-group texts and explicit reporting-category mapping. | Same invoice amount as the headline. Assign one reporting supplier and category per invoice item. Keep unknown/unassigned categories visible. Invoice detail totals must reconcile to the headline. |
| Non-managed spend amount and percentage | Full invoice population, including invoices without PO references. Use a supplemental synthetic file if the supplied extractor structure excludes those invoices. | Net invoice spend without PO reference / all in-scope net invoice spend. Posting-date period. Supplemental invoices must be counted exactly once. |
| Total PO value; on/off-contract value and shares | C_PurchaseOrderItemDEX; C_PurchaseOrderDEX when header attributes are needed. PO/item keys, supplier, company code, purchasing organization, creation date, net value/currency, contract/item reference, deletion/cancellation indicators. | Sum eligible PO-item net value by PO creation period. Split by presence of a valid contract reference; retain unknown classification separately if necessary. Ratios use total eligible PO value. Receipt and invoice amounts are not added to ordered value. |
| Potential contract leakage | Above PO data; C_PurchaseContractDEX; C_PurchaseContractItemDEX; C_PurchaseContractHistoryDEX. Contract/item keys, supplier, purchasing organization, material, contract type, validity, target quantity and release quantities/dates. | For this POC, flag an off-contract PO item when at least one applicable quantity contract matched supplier, purchasing organization and material, was valid on PO creation date and had sufficient remaining quantity. Count the PO item once even if several contracts match. Sum flagged PO-item net value. Overlapping subset of off-contract value, not savings. This is a simplified proposed rule, not verified F0681 parity. |
| Unused active contracts | Contract header/items and release history. | Distinct contracts active and usable at the as-of date, with no qualifying non-reversed releases during the selected period. Count contracts, not items. New contracts with no releases are included under this proposed rule. |
| Overdue open PO value and share | C_PurchaseOrderItemDEX; C_PurOrdScheduleLineDEX; C_PurchaseOrderHistoryDEX; PO header context as needed. Schedule quantities/dates, net receipts/reversals, PO prices and price units, completion/deletion status, units and currencies. | For open eligible schedules due before the as-of date, sum remaining quantity × PO net price / price unit, after unit normalization. Share = overdue value / all open eligible delivery value at that date. For the initial POC use quantity-based material POs and label this scope; service/limit items require a separately agreed amount-based rule. Include older open POs. |

## Required source-shaped transaction files

Generate coherent synthetic files based on:

1. C_PurchaseOrderDEX — header and document context, if supplied; do not duplicate a header measure in item totals.
2. C_PurchaseOrderItemDEX — PO items.
3. C_SupplierInvoiceDEX — invoice headers.
4. C_SupplierInvoiceItemDEX — invoice items covered by the supplied structure.
5. C_PurOrdScheduleLineDEX — scheduled deliveries.
6. C_PurchaseOrderHistoryDEX — distinguish receipt, invoice and reversal event populations.
7. C_PurchaseContractDEX — contract headers.
8. C_PurchaseContractItemDEX — contract items and applicability.
9. C_PurchaseContractHistoryDEX — release activity.
10. Synthetic_NonPOInvoiceItems and associated headers, only if needed to cover non-PO spend outside the supplied invoice structure.

Separate files need not be generated merely to duplicate information already supplied by another file. If any source file is unavailable, a clearly labeled synthetic supplemental structure can support the POC; document the unvalidated production mapping and do not claim it is an extract of that CDS view.

## Real dimension inputs

| Dimension | Preferred candidate or accepted input |
|---|---|
| Supplier | I_Supplier. I_BusinessPartner may be used with an explicit supplier-to-BP mapping; do not assume equal identifiers. |
| Company code | I_CompanyCode |
| PO type | I_PurchaseOrderType and I_PurchasingDocumentTypeText; filter texts to one reporting language and preserve document-category context. |
| Material/product group | I_ProductGroupText_2 or the equivalent supplied group list; an explicit mapping supplies broader reporting categories. |
| Purchasing organization | I_PurchasingOrganization or the equivalent supplied list |
| Purchasing group | I_PurchasingGroup |
| Plant | I_Plant when retained in the generated transactions |

Transparent-table lists with the equivalent keys and descriptions are acceptable for the POC. Use supplied organizational mappings and supplier/category examples to constrain combinations. If not supplied, document synthetic assignment assumptions rather than claiming those relationships are real. Do not infer a business-unit hierarchy from company-code names: use a supplied mapping or use company code as the filter.

## Dates, volume and acceptance checks

- Exactly 5,000 unique invoice items across the native-shaped and supplemental invoice populations combined.
- Invoice posting window: September 1, 2024–August 31, 2026. Default comparison: September 2025–August 2026 versus September 2024–August 2025. Display fiscal/calendrical meaning explicitly; this is a rolling 12-month comparison, not fiscal YTD.
- Default open-order snapshot: August 31, 2026. Historic open-order comparison is outside this first-page scope.
- Generate supporting PO/contract dates before the window where required. Future scheduled deliveries after the snapshot are valid; posted events after the snapshot are not included.
- Top three suppliers approximately 55% of invoice spend; next seven approximately 30%; others approximately 15% if there are around 20 supplied suppliers. Unequal members within groups and different document-count shares.
- Preserve complete keys, currencies and identifier padding. Header keys must not be assumed unique without fiscal year/source context.
- Use separate source-grain facts. Join descriptive information many-to-one. Receipt history and invoice history must not duplicate invoice fact amounts.
- Keep posted credits/reversals; model signs once. A retained original posting and its reversal must net correctly. Do not both remove the original and retain a negative reversal.
- Model at least some partial receipts, multiple invoices against a PO item, multiple schedules and POs not yet invoiced. Include off-contract, potential leakage and active-unused contract cases.
- If source history has no schedule-line reference, provide an explicit synthetic allocation bridge, assigning net receipts to earliest due schedules first. Reconcile it to PO-item receipts; do not present it as a standard source field.
- Sum at least to invoice supplier/category/month and company code; reconcile detail to headline and on/off-contract classifications to their PO population. Leakage must be a subset of off-contract value. Check open value, overdue value and zero denominators.
- For a multicurrency POC, retain document amounts/currencies plus reporting-currency amounts and an explicit synthetic rate table. Never sum unconverted amounts from different currencies.

## Copilot prompt — copy from here

Create the complete synthetic transaction package for an SAC Purchasing Spend Overview using the attached transaction samples and real dimension lists. Use the metric-to-source checklist above as the acceptance criteria.

Preserve supplied CDS column names, data types and keys. Keep real supplier, company-code, PO-type, purchasing-organization/group, plant and material-group codes/descriptions unchanged. Preserve identifiers and leading zeros as text. Use valid supplied organizational and supplier/category relationships. Document any assumed relationships; do not label them as actual master data.

Generate exactly 5,000 invoice items posted from September 1, 2024 through August 31, 2026, plus all related invoice headers, PO headers/items, schedules, history events and contract headers/items/releases required to support them. Counts in related files should follow business relationships, not each equal 5,000. Use new synthetic document identifiers consistent across files. Retain real document IDs only if explicitly requested.

Create a realistic, uneven distribution. With approximately 20 suppliers, allocate about 55% of net invoiced spend to three suppliers, 30% to the next seven and 15% to the remainder, varying shares within groups. Adapt to the actual supplier count. Use different document-count and spend rankings, frequent small purchases, occasional larger purchases, supplier/category specialization, monthly seasonality and differing year-over-year trends. Avoid equal splits, uniform random amounts and identical growth rates. Use plausible outliers sparingly.

Support every first-page measure: invoice spend/trend/comparable-year change, supplier/category spend, invoice detail, non-PO spend, ordered PO value, on/off-contract PO shares, potential contract leakage, unused active contracts, and overdue open delivery value/share. Preserve the distinct date bases and populations in the checklist. Do not add receipts, POs or payments to invoice spend.

Include partial receipts and invoices, multi-item POs, multiple schedules, signed credits/reversals, open and fully completed POs, overdue and not-yet-due deliveries, contract releases and active unused contracts. Keep amounts, suppliers, currencies, references and dates coherent across files. Supporting PO/contract dates may precede the invoice window. Snapshot open orders at August 31, 2026; for the initial overdue measure use quantity-based material POs, with that scope explicitly labeled.

For potential leakage, use the checklist's quantity-contract rule and count each affected PO item only once. Do not invent a monetary savings measure. Keep invoice-without-PO records available: if the supplied invoice extractor cannot represent them, produce clearly labeled supplemental invoice/header structures within the combined 5,000-item total. If another supplied structure lacks information required by a metric, add a separate documented synthetic supplement rather than silently adding invented standard-CDS columns. Proceed with the POC and identify the production mapping gap.

Use the same source data for all outputs. In addition to source-shaped CSVs, generate separate SAC-ready datasets at these grains: invoice item; PO item; open delivery schedule at the snapshot; contract header summary at the snapshot. Put potential-leakage flags on PO items. Preserve any generated receipt-allocation bridge separately. Resolve descriptions and category mappings without multiplying amounts. Include appropriate date basis, currency and eligibility fields. Do not combine all four grains into one flat table.

Use a reproducible script and provide downloadable UTF-8 CSVs labeled Synthetic, the generation script, a data dictionary, and a reconciliation report. Document every derived/supplemental field, mapping and synthetic exchange rate. Verify 5,000 unique invoice items, complete relationships, valid dimension members, date logic, credits/reversals and totals. Report monthly totals, comparable-year changes, supplier spend shares/document counts and scenario counts. Demonstrate that source-shaped files and SAC-ready outputs reconcile, that leakage is contained in off-contract value, and that overdue value does not exceed open eligible delivery value. Deliver the files, not just sample rows or a proposed approach.
