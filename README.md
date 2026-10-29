# Hambisa — Motorcycle Delivery Platform

Hambisa (isiZulu: "to send on its way") is the working name; confirm trademark availability before launch. The product specification in `docs/` still uses its original placeholder name, SwiftRide.

## Applications

| Application | Platform | Status |
|---|---|---|
| Customer app | React Native (iOS, Android) | UI designed |
| Rider app | React Native (iOS, Android) | UI designed |
| Back office | Angular | UI designed |
| Merchant portal | Angular (phase 2) | UI designed |
| Backend services | C# / ASP.NET Core microservices | Not started |

## Repository layout

```
docs/     Product specification
design/   UI designs, one file per screen
```

## Design screens

| File | Screen |
|---|---|
| `Main.dc.html` | Foundations — colour, type, status chips, controls |
| `Cus01-SignIn.dc.html` | CUS-01 Sign in · verify code |
| `Cus02-Home.dc.html` | CUS-02 Home & booking |
| `Cus03-Review.dc.html` | CUS-03 Parcel details & review |
| `Cus04-Payment.dc.html` | CUS-04 Payment & confirmation |
| `Cus05-Tracking.dc.html` | CUS-05 Live tracking |
| `Cus06-History.dc.html` | CUS-06 Deliveries list |
| `Cus07-Completed.dc.html` | CUS-06 Delivery completed detail |
| `Rid01-Onboarding.dc.html` | RID-01 Onboarding & verification |
| `Rid02-Offer.dc.html` | RID-02 Availability & job offer |
| `Rid03-Collect.dc.html` | RID-03 Collection |
| `Rid04-Active.dc.html` | RID-04 Active delivery (offline) |
| `Rid05-Proof.dc.html` | RID-05 Delivery proof |
| `Rid06-Earnings.dc.html` | RID-06 Earnings |
| `Bo01-Dashboard.dc.html` | BO-01 Operations dashboard |
| `Bo02-Orders.dc.html` | BO-02 Order management |
| `Bo03-Dispatch.dc.html` | BO-03 Dispatch control centre |
| `Bo04-Riders.dc.html` | BO-04 Rider management |
| `Bo05-Payments.dc.html` | BO-05 Payments & refunds |
| `Bo06-Pricing.dc.html` | BO-06 Pricing & service areas |
| `Bo07-Support.dc.html` | BO-07 Support cases |
| `Bo08-Reports.dc.html` | BO-07 Reports & exports |
| `Mer01-BulkBooking.dc.html` | Merchant portal · Bulk booking |

`design/canvas.json` holds the canvas layout. The `.dc.html` files are Claude Design canvas components; sample data in them is illustrative, and bracketed values such as `[LIMIT]` are policy decisions still to be confirmed.
