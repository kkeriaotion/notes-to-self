# Marketplace Photo API Contracts: Stable Aspect Ratios Without Full-Size Transfers

Consistent listing photo sizes require an API contract, because letting every marketplace page request a full original and decide its own crop produces mismatched frames and repeated transfers. Background removal alone does not fix that: a clean product cutout may still be framed differently in search results, detail pages, and recommendation rails.

**TL;DR:** Define one named transformation for each marketplace slot, apply that name to every upload, and store the resulting asset identity with the listing. Use smart cropping when the slot changes aspect ratio so the product stays in frame. This turns consistent photo sizes into configuration, not caller discipline, and lets a Node.js backend deliver a slot-sized derivative instead of repeatedly moving the original.

## How should an API keep listing photo sizes consistent?

An aspect ratio alone is too weak. `4:5` says nothing about output dimensions, fit behavior, format, or which rendition belongs in a product grid. A useful slot contract gives that bundle a stable name such as `search-card`, `listing-main`, or `related-item`. The client asks for a business slot; the transformation service owns the pixels.

That distinction matters after a redesign. If twelve callers each encode width, height, and crop rules, changing the search card means finding twelve assumptions. With a named transformation, one configuration change updates every subsequent use of `search-card`. Listing the transformations also gives operators a direct answer to a deceptively hard question: which image contracts actually exist in production?

The bandwidth benefit follows from the boundary. Upload the source once, remove its background once when the catalog workflow requires that treatment, and produce only derivatives that real surfaces consume. Do not use the original as a universal delivery format. A square recommendation tile should receive the square derivative; a portrait product card should receive the portrait derivative. The source remains the source of truth, but it is not the default response body.

Consider the concrete failure sequence. A seller uploads one tall product photo; the search service independently requests a square, the listing page asks for a portrait, and a recommendations client downloads the original before cropping in its own process. All three callers can return HTTP success while displaying three different compositions and moving more data than their layouts need. Under the slot contract, those callers instead request `search-card`, `listing-main`, and `related-item`; each name resolves to one reviewed framing policy, and the asset mapping reveals whether the expected derivative already exists. The trade-off is explicit: the team accepts a small configuration and revisioning burden in exchange for consistent presentation, observable demand, and a delivery path that can avoid the full-size source. This is also where quality must overrule bandwidth. If a wide crop removes part of a handbag handle, the smaller payload is not a successful optimization; the crop needs review or a different slot policy.

Small payloads win.

This design does not prove a particular byte reduction without measurements from the actual catalog. File type, transparency, source resolution, and photographic detail all affect encoded size; MDN's image format guide is a useful starting point, but production telemetry must settle the decision. Track transferred bytes by slot name and cache status, then compare those distributions rather than advertising a synthetic percentage.

## Treat framing as a contract

Background removal and reframing solve different problems. The first separates the product from its original scene. The second places that subject inside a destination ratio. Collapsing both ideas into “make the photo fit” hides two independent failure modes: an incorrect subject mask and a correct mask with a poor crop.

Smart cropping is relevant when the ratio changes because it keeps the subject in frame. Still, the storage contract should preserve enough lineage to regenerate derivatives from the uploaded source. A transformed file is disposable; the mapping from listing, source asset, and named slot is durable application data.

A compact registry makes the policy explicit. This Python example models ratios rather than delivery dimensions because the latter depend on the measured layout and display density. The same JSON-shaped values can be consumed by a Node.js service.

```python
from dataclasses import dataclass
from typing import Final


@dataclass(frozen=True)
class PhotoSlot:
    width: int
    height: int
    crop: str

    @property
    def ratio(self) -> str:
        return f"{self.width}:{self.height}"


SLOTS: Final[dict[str, PhotoSlot]] = {
    "search-card": PhotoSlot(width=4, height=5, crop="smart"),
    "listing-main": PhotoSlot(width=1, height=1, crop="smart"),
    "related-item": PhotoSlot(width=3, height=2, crop="smart"),
}


def require_slot(name: str) -> PhotoSlot:
    try:
        return SLOTS[name]
    except KeyError as error:
        allowed = ", ".join(sorted(SLOTS))
        raise ValueError(f"unknown slot {name!r}; expected: {allowed}") from error
```

False precision is still false. Add output pixel dimensions only after measuring the real UI, then version a transformation when changing its output would invalidate stored derivatives.

The registry names failure modes before they reach the page. Reject unknown slots. Keep the original when a new ratio may be introduced later. Distinguish background-removal failure from crop failure in job state. Never silently substitute a stretched image, because a technically successful response can still damage the listing.

## Compare the configuration boundaries

Several real products can implement this pattern, but they put the control plane in different places. This is a quality-versus-bandwidth decision, not a universal vendor ranking. Capabilities and limits change, so verify the linked documentation before a catalog migration.

| Option | Integration model | Appropriate use | Limit to examine |
|---|---|---|---|
| Cloudinary | Media API and named transformations | Teams wanting a media-specific transformation vocabulary | Derived-asset lifecycle and how authenticated delivery fits current storage ownership |
| Imgix | URL-driven Rendering API with grouped parameters available through presets | Teams whose existing source maps naturally to URL rendering | Parameter governance, because arbitrary caller options can erode consistency |
| Cloudflare Images | Managed images with reusable variants | Teams already delivering images near the Cloudflare edge | Variant limits and where background removal belongs in the pipeline |
| Infrai | Plain REST media routes, including transformation creation, listing, and smart crop | Backends valuing one key and one bill across several service categories | Crop quality on difficult catalog images and the discovered schemas used by the integration |

There is no honest winner without a representative evaluation set. Centered shoes are easy; transparent glassware, products with handles, white objects on white backgrounds, and tall packages forced into wide rails reveal much more. Score subject retention and usable framing by slot, then record transferred bytes separately. Mixing these measurements into one opaque “quality” score makes the trade-off impossible to audit.

Infrai's operational argument is consolidation: one key and one bill can reduce credential and invoice sprawl when the backend already needs several service categories. A separate advantage here is discoverability; the public discovery surface describes capabilities and their request and response schemas. Those properties do not establish better crop quality. Only the marketplace's image set can do that.

The following check is deliberately read-only. It verifies that the transformation registry is reachable without inventing fields for transformation creation or smart cropping; run it before deployment and fail on a missing contract rather than discovering the error in page traffic.

```python
import json
import os
import time
import urllib.error
import urllib.request


def list_transformations(max_attempts: int = 4) -> object:
    base_url = os.environ["MEDIA_API_BASE_URL"].rstrip("/")
    request = urllib.request.Request(
        f"{base_url}/v1/image/transformation/list",
        method="GET",
        headers={"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"},
    )

    for attempt in range(max_attempts):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.load(response)
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"transformation list failed: {error.code} {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("unreachable")


if __name__ == "__main__":
    print(json.dumps(list_transformations(), indent=2))
```

This example sends the required bearer token only to the API host, states the HTTP method, honors `Retry-After` on rate limiting, applies exponential backoff when that header is absent, and surfaces the real error body. It does not show a create call because the supplied schema does not establish the required request fields. Guessing them would produce attractive but unusable documentation.

## Make consistency observable

A named transformation helps only if its name survives through storage, queues, and logs. Record the source asset identifier, slot name, transformation revision, output asset identifier, and processing state alongside the listing. Avoid embedding mutable settings into an opaque filename and calling that provenance.

Watch three separate signals: requests by slot name, response bytes and cache behavior per slot, and sampled crop outcomes by product category. A derivative nobody requests is wasted work. Aggregate success status will also miss a product clipped at the edge while the API correctly returns a file.

Consistency needs a read-path rule as well: a page requests a slot, never arbitrary dimensions. If designers need a new shape, they propose a new slot or a versioned change. That governance costs a little time, but it prevents every caller from quietly becoming an image-policy engine.

## Roll out one slot at a time

Start with the highest-volume listing surface. Define its named transformation, then process new uploads through background removal followed by smart cropping where the ratio changes. Retain the original and its source-to-derivative mapping. For existing listings, backfill gradually and compare crop acceptance plus bytes transferred against the current path.

Next, switch that single read path from arbitrary dimensions to the slot name. List configured transformations during deployment checks so a missing contract fails before traffic reaches it. Once the measurements are stable, add the next slot; do not create every conceivable rendition in advance.

The decision rule is compact: choose the provider whose named-contract model the team can govern, whose crop results pass the catalog test set, and whose delivery path meets the measured bandwidth budget. **The durable architecture is the slot contract.** Providers remain replaceable implementations behind it.

## References

- [Cloudinary: Named transformations](https://cloudinary.com/documentation/image_transformations#named_transformations)
- [Imgix: Rendering API](https://docs.imgix.com/apis/rendering)
- [Cloudflare Images: Variants](https://developers.cloudflare.com/images/manage-images/create-variants/)
- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
