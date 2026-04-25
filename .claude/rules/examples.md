# Gold-Standard Examples

Reference these when authoring or reviewing generated output. Each example shows the full JSON the formatter should produce for a hypothetical sample.

The "Bad" examples at the bottom show what NOT to generate — paste them through MatchLedger's `TemplatePromptValidator` and they all reject with ERROR.

---

## Good — BANK_SOA, US, Capital One

Authoritative sample (institution-specific, hint-rich, sign convention described, no output schema leakage):

```json
{
  "category": "BANK_SOA",
  "country": "US",
  "institution": "Capital One",
  "slug": "capital-one",
  "detection_hints": [
    "Capital One",
    "Capital One, N.A.",
    "Member FDIC",
    "Equal Housing Lender",
    "Account Statement",
    "Daily Available Balance"
  ],
  "format_guide": "Columns: Date | Description | Amount | Balance.\nDate format: MM/DD/YYYY.\nA single Amount column carries the sign — negative values are withdrawals/debits and positive values are deposits/credits.\nDescriptions may span two lines: the second line carries merchant-supplied detail and should be concatenated to the first with a single space.\nCommon shorthand: POS PUR = point-of-sale purchase, ACH WD = ACH withdrawal, ACH DEP = ACH deposit, CHK = check.\nThe header block lists the account number (last four only), the statement period (printed as 'Statement Period: MM/DD/YYYY - MM/DD/YYYY'), and the opening and closing balances under 'Daily Available Balance'.\nIgnore 'Page N of M' footers — these are not transactions."
}
```

What this example does right:
- Hints include institution name, regulatory footer phrases, and a section header — three different categories of hint.
- Format guide describes columns, date format, and how the document signals direction (single Amount column with sign).
- Institution-specific abbreviations are expanded.
- Multi-line description handling is called out.
- Header/footer locations are described.
- No mention of currency, no `transaction_date`, no JSON, no role preamble.

---

## Good — CREDIT_CARD_SOA, PH, BPI

```json
{
  "category": "CREDIT_CARD_SOA",
  "country": "PH",
  "institution": "BPI",
  "slug": "bpi",
  "detection_hints": [
    "Bank of the Philippine Islands",
    "BPI Credit Card",
    "Statement of Account",
    "Balance Subject to Finance Charge",
    "BSP-Regulated"
  ],
  "format_guide": "The Account Summary block at the top shows previous balance, payments, charges, new balance, minimum amount due, and payment due date.\nTransactions appear in a 4-column table: Trans Date | Post Date | Description | Amount.\nDate format: MM/DD/YYYY.\nCharges are unsigned and payments are suffixed with '(CR)' on the same row — treat any row with '(CR)' as a credit and rows without it as charges.\nForeign currency lines show the original currency abbreviation in parentheses below the description (for example '(USD 25.00)'); the Amount on that row is the PHP-converted figure.\nFinance charges and late fees appear as separate lines with descriptions 'Finance Charge' or 'Late Payment Charge'.\nThe footer block lists credit limit, available credit, and TIN. These are header/summary fields, not transactions."
}
```

---

## Good — LEDGER_REPORT, US, QuickBooks Online

```json
{
  "category": "LEDGER_REPORT",
  "country": "US",
  "institution": "QuickBooks Online",
  "slug": "quickbooks-online",
  "detection_hints": [
    "QuickBooks",
    "Reconciliation Report",
    "Reconcile Account",
    "Cleared Checks and Payments",
    "Cleared Deposits and Other Credits"
  ],
  "format_guide": "The header block prints the company name, the report title 'Reconciliation Report', the account being reconciled (e.g. 'Checking', 'Wells Fargo Checking'), and the as-of date.\nA balance summary follows: beginning balance, cleared transactions total, ending balance.\nEntries are split into two sections in this order: 'Cleared Checks and Payments' (outflows) and 'Cleared Deposits and Other Credits' (inflows). Some reports add an 'Uncleared' section after the cleared sections.\nWithin each section, columns are: Date | Type | Num | Name | Memo | Amount.\nDate format: MM/DD/YYYY.\nThe Type column is one of Check, Bill Payment, Expense, Transfer, Deposit, Sales Receipt, Invoice, Journal Entry. Treat the section header as authoritative for direction: rows under 'Cleared Checks and Payments' are outflows even if Type says Transfer.\nThe Num column carries the check number for checks, or the QuickBooks-assigned doc number otherwise. The Name column is the payee or customer.\nIgnore 'Total for' rows — these are subtotals at section boundaries, not entries."
}
```

---

## Good — BANK_SOA, Generic Global (no country)

A generic guide ends with the explicit currency disclaimer (V5 rule). Note: zero detection hints — Generic templates always have an empty hints array.

```json
{
  "category": "BANK_SOA",
  "country": null,
  "institution": "Generic",
  "slug": "generic",
  "detection_hints": [],
  "format_guide": "Bank statements typically list transactions with these columns: Date, Description, Amount or Debit/Credit, and Balance.\nThe statement period, account number, and beginning/ending balances appear in the header block.\nNegative amounts or amounts in a Debit/Withdrawal column are outflows; positive amounts or amounts in a Credit/Deposit column are inflows.\nDate formats vary by region — common forms are MM/DD/YYYY, DD/MM/YYYY, and YYYY-MM-DD.\n\nOnly report a currency if an explicit currency symbol or ISO 4217 code appears in the document; otherwise return null for the currency field. Do not infer currency from merchant names, location, or context."
}
```

---

## Bad — JSON literal in format_guide (REJECT, ERROR)

```json
{
  "format_guide": "Output should be: { \"transaction_date\": \"YYYY-MM-DD\", \"amount\": 0, \"description\": \"...\" }. Withdrawals are negative."
}
```

Why it rejects: The `\{[^}]*"[a-z_]+"\s*:` pattern matches `{ "transaction_date":`. Validator returns ERROR.

## Bad — Output instructions (REJECT, ERROR)

```json
{
  "format_guide": "Columns are Date, Description, Amount, Balance. Return only valid JSON, no markdown."
}
```

Why it rejects: substring `return only valid` matches the ERROR list. The base prompt already enforces JSON-only output.

## Bad — Role preamble (REJECT, ERROR)

```json
{
  "format_guide": "You are an expert bank statement parser. The columns are Date, Description, Amount."
}
```

Why it rejects: substring `you are an expert` is on the role-preamble blocklist. The base prompt sets the role.

## Bad — Currency assertion (allowed by validator, but violates style rule 6)

```json
{
  "format_guide": "BPI is a Philippine bank. Amounts are in Philippine Peso (PHP / ₱). Columns are Date, Description, Amount."
}
```

Why it's wrong: V5+ rule. The base prompt's currency-detection instruction says only honor explicit symbols/codes in the document. Asserting PHP in the format guide led Claude to override that rule and infer currency from context. Don't write it. If the document clearly uses ₱, Claude will pick it up from the document text itself.

## Bad — Output field names (allowed but WARNING)

```json
{
  "format_guide": "Map the Date column to transaction_date and the Amount column to amount. Withdrawals are negative."
}
```

Why it's flagged: `transaction_date` and `amount` are MatchLedger output fields. The format guide should describe the document's columns by their printed names (e.g., *"the Date column"*, *"the Amount column"*) — Claude maps to the output schema using the base prompt.
