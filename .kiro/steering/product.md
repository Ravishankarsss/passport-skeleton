# Product

Passport Skeleton is a lean template for Digital Product Passports.
A passport is a public record of what a product is made of, who made
each part, and what proof exists.

## Core idea
One config file describes an industry: its fields, proof documents and
risk rules. The skeleton turns that config into a working app. Adding
an industry must never require changing skeleton code.

## Apps built on the skeleton
- Garment passport: for export garment factories.
- Software passport: lists libraries, versions and licences.

## Rules
- Never invent or guess data. A missing fact is shown as "missing".
- Every field has a status: proven, claimed or missing.
- "Proven" requires a linked proof document.
- Each product gets a readiness score from 0 to 100.
- Each product has a public page reachable by QR code.

## Words we use
- Passport: the full record for one product.
- Field: one fact, such as fibre content or a library version.
- Proof: a document that supports a field.
- Config: the file that defines one industry.