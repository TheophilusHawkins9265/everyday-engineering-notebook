# Marketplace PDF Forms: 2 Print Layout Commitments Beyond HTML Rendering

| Template ownership | Default path | Main reason |
|---|---|---|
| Your team owns the PDF form | Fill the existing fields, then flatten the result | Preserve the approved pages while removing editable controls |
| Another party owns the PDF form | Treat the file as an external contract | Detect template drift before producing final documents |
| Your team owns only the content | Generate pages from a controlled layout model | Stop pretending an unknown form is a stable application interface |

**TL;DR:** For a marketplace document that must be filled and flattened, prefer the existing PDF form when your team owns and versions that template. PDF generation is harder than browser rendering because the output is a finished, page-bound artifact: field coordinates, font resources, appearance data, page boxes, and the final non-editable result all have to agree. A browser can keep laying out a living document. A PDF producer has to commit.

The recommendation is narrow on purpose. Own the template, and a fill-then-flatten pipeline gives you the smallest review surface. Do not own it, and the same approach becomes brittle unless you validate every incoming revision. Template ownership matters more than which SDK has the shortest sample snippet.

## Why is PDF print layout harder than HTML rendering?

HTML begins with structure and lets a browser resolve the current presentation. The viewport can change. Text can reflow. A font substitution may move a line, and the page can still remain usable because the browser computes layout again.

A marketplace PDF form has a different job. It may need to preserve an approved disclosure, place a seller name inside a fixed rectangle, keep a signature area on page 3, and arrive as a document whose form controls can no longer be edited. ISO 32000-2 specifies PDF 2.0, a paginated document format built from graphics and other objects. Producing one means resolving those objects into a coherent file, not handing the next renderer an open-ended layout problem.

Commitment is the hard part.

Consider a long legal name. In HTML, a flex or grid container may grow, wrap, or push later content down. In a fixed form, wrapping may cover the next label, shrinking may make the disclosure unreadable, and clipping may remove legally important text. None of those outcomes is fixed by a successful file write. The generator needs an explicit overflow policy for each field.

Flattening adds another boundary. The visible field value must survive after the interactive control is removed from the workflow. A pipeline that merely assigns a value has not proved that a plain PDF viewer will show the same final page. The artifact, not the in-memory object, is the test target.

## Criterion 1: Who can change the template?

A form owned by the same team as the generator can act like a versioned interface. Give every template an immutable identifier. Keep field names, page count, expected page boxes, and required fields in a manifest beside it. Review a template change with the code that consumes it. This is config, but it earns its keep: one small contract replaces scattered coordinates and guesses.

Third-party forms reverse the risk. A visually minor revision can rename a field, move a widget, add a page, or alter the resources used to display a value. The integration can still run and produce the wrong document. That failure mode is worse than a clean rejection.

So reject unknown revisions before filling them. Compare an approved template fingerprint and structural expectations, then route a mismatch for review. Do not silently fall back to old coordinates. A marketplace team may be tempted to accept anything that looks close because onboarding speed is visible; the malformed final agreement shows up later, when recovery is expensive.

The useful decision rule is blunt: **if you cannot version the template, version your acceptance criteria for it.** Ownership determines where that contract lives, not whether you need one.

## Criterion 2: Can you prove the flattened artifact?

The happy-path benchmark is almost meaningless. Time-to-first-file says little about correctness. Measure the stages separately: template validation, field mapping, value assignment, flattening, serialization, and verification. Record durations and failure categories, but avoid storing document values in logs. Marketplace forms routinely carry names, addresses, tax details, or signatures.

Verification needs layers. First, reopen the emitted bytes with a parser rather than trusting the object that wrote them. Confirm the expected page count and the absence of interactive fields after flattening. Next, extract or inspect content according to the requirements of the document. Finally, render representative pages and compare them with approved baselines using tolerances chosen for the renderer and environment. Structural checks catch missing controls. Render checks catch clipped text and displaced appearances. Neither replaces the other.

Use adversarial fixtures, not a single tidy seller. Include an empty optional field, the longest accepted business name, non-ASCII text allowed by the product, a multiline address, and values at every declared length boundary. Keep at least one fixture for each template version still accepted in production.

Operationally, preserve three identifiers: input template version, generation build, and output content hash. Those identifiers let an engineer reproduce a disputed artifact without turning logs into a second document store. Set retention and access controls around the generated file according to its data, not according to the fact that its extension is `.pdf`.

## A narrow TypeScript boundary

PDF libraries expose different object models, so application code should not leak one throughout the marketplace service. Keep the adapter small. The interface below makes validation and post-write verification mandatory while leaving the underlying implementation unspecified.

```ts
type TemplateId = string;

type SellerAgreementData = {
  sellerLegalName: string;
  marketplaceName: string;
  mailingAddress: string;
};

type FinalizedPdf = {
  bytes: Uint8Array;
  sha256: string;
  pageCount: number;
};

interface PdfFormAdapter {
  assertApprovedTemplate(template: Uint8Array, id: TemplateId): Promise<void>;
  fillAndFlatten(
    template: Uint8Array,
    data: SellerAgreementData,
  ): Promise<Uint8Array>;
  verifyFinalized(
    bytes: Uint8Array,
    expectedTemplate: TemplateId,
  ): Promise<FinalizedPdf>;
}

async function buildSellerAgreement(
  adapter: PdfFormAdapter,
  template: Uint8Array,
  templateId: TemplateId,
  data: SellerAgreementData,
): Promise<FinalizedPdf> {
  await adapter.assertApprovedTemplate(template, templateId);

  const bytes = await adapter.fillAndFlatten(template, data);
  return adapter.verifyFinalized(bytes, templateId);
}
```

This boundary refuses a common shortcut: returning bytes directly from `fillAndFlatten` to storage. The second parse is deliberate. It tests the serialized artifact and gives one place to enforce page count, interactive-field, and hashing rules.

Keep field mapping inside the adapter or a versioned manifest. Adding a generic configuration engine for every possible form usually creates more glue than it removes. Three explicit mappings that fail loudly are easier to review than a clever mapper that accepts ambiguous input.

## When is controlled page generation the better runner-up?

Generate from a controlled layout model when your team owns the content and presentation but does not need to preserve an externally approved form. This path is also stronger when sections vary substantially in length, tables can span pages, or localization changes text volume enough that fixed field rectangles become the dominant risk. You still have to resolve pagination, fonts, overflow, headers, footers, and testing. You have merely moved those decisions into a layout system you control.

The limitation of fill-and-flatten is its dependence on a stable form. It is a poor fit for reports with optional sections or tables of unknown length. The trade-off is explicit: the existing form reduces layout freedom in exchange for preserving an approved page design; controlled generation accepts more pagination work in exchange for accommodating variable content.

Do not convert HTML to PDF by reflex. It can be a reasonable implementation detail for content that already follows browser layout rules, but it does not erase print constraints. Define page size, margins, break behavior, font availability, and the rendering environment. Then test the emitted PDF as a PDF.

The existing-form path wins when fidelity to an owned, approved template is the primary constraint. Controlled generation wins when variable content and layout evolution are primary. **Choose by who owns change.** Library syntax comes later.

## Sources

References:

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
