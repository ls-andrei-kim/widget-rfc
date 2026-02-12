# 80. Lightspeed Reservation Widget

**Owner(s):**

- Vladislav Khripkov, Andrei Kim

**Created:** 2026-02-12

**Status:** Draft

## What are you trying to solve?

Guests want a simple way to book reservations directly from the merchant website without leaving the page. The current process involves redirecting to our reservation site (`lightspeed.app/reservation/{merchant-id}`), which creates several problems:

- **High drop-off rates**: Users abandon the booking flow when redirected to external sites
- **Fragmented user experience**: Breaking the merchant's website flow reduces conversion
- **Competitive disadvantage**: Competitors like OpenTable, Resy, and SevenRooms offer embeddable widgets

Merchants are currently using third-party reservation widgets because we don't provide an embeddable solution. This proposal aims to provide an embeddable widget that keeps users on the merchant's website throughout the entire booking flow.

## What are you proposing?

We will develop a customizable, embeddable JavaScript widget that merchants can easily integrate into their websites. This widget will communicate with our existing Reservation API to check availability and process bookings.

### Key Features

- **Booking Form**: A simple, multi-step form for guest information
- **Instant Confirmation**: Immediate booking confirmation with email notification
- **Customization**: Merchants can configure widget appearance and behavior
- **Responsive Design**: Works seamlessly on desktop and mobile devices
- **Accessibility**: WCAG 2.1 AA compliant for inclusive user experience

### Technical Decisions

Based on analysis of competitor solutions (OpenTable, Resy, SevenRooms, TheFork) and technical feasibility, we've chosen:

- **Delivery Method**: Iframe pointing to a new widget-specific page
- **Configuration**: Hybrid approach (backend defaults + URL query parameter overrides)
- **Browser Support**: Modern browsers (Chrome, Firefox, Safari, Edge - last 2 versions), mobile responsive, WCAG 2.1 AA compliant

### Widget Delivery

**Chosen Approach: Iframe with New Widget-Specific Page**

The widget will be delivered in two parts:

1. **Loader Script**: A lightweight JavaScript file hosted on our CDN (S3 + CloudFront)
2. **Widget Application**: An iframe pointing to a new widget-specific page

**Implementation:**

Merchants will add a single script tag to their website:

```html
<script src="https://reservations.lightspeedhq.com/widget-loader.js?venueId=123&theme=dark"></script>
```

The loader script will:

1. Read configuration from URL parameters and/or `window.lsk_reservation` object
2. Create an iframe pointing to `https://reservations.lightspeedhq.com/reservation/{venue-id}/widget`
3. Inject the iframe into the merchant's page
4. Handle iframe resizing and communication via postMessage API

**Why Iframe Approach:**

- **Security**: Better isolation between merchant site and widget (no XSS risks)
- **Faster Development**: Reuses existing reservation infrastructure
- **Easier Maintenance**: Widget updates don't require merchant code changes
- **Proven Pattern**: Used successfully by competitors (OpenTable, TheFork, Zenchef)

### System Architecture

```mermaid
graph TB
    subgraph "Merchant Website"
        MerchantPage[Merchant Web Page]
        LoaderScript[Widget Loader Script]
        WidgetIframe[Widget Iframe]
    end

    subgraph "Lightspeed Infrastructure"
        CDN[CDN<br/>CloudFront + S3]
        ReservationWeb[Reservation Website<br/>Widget Page]
        ReservationAPI[lsk-reservation-service<br/>API]
        Backoffice[Backoffice<br/>Configuration]
    end

    subgraph "Guest"
        User[Guest/Customer]
    end

    User -->|1. Visits| MerchantPage
    MerchantPage -->|2. Loads| LoaderScript
    LoaderScript -->|3. Fetches from| CDN
    LoaderScript -->|4. Creates & injects| WidgetIframe
    WidgetIframe -->|5. Loads content from| ReservationWeb
    ReservationWeb -->|6. Fetches config| Backoffice
    ReservationWeb -->|7. API calls| ReservationAPI
    ReservationAPI -->|8. Email notification| User

    style CDN fill:#e1f5ff
    style ReservationAPI fill:#fff4e1
    style Backoffice fill:#f0f0f0
    style WidgetIframe fill:#e8f5e9
```

### Configuration

**Chosen Approach: Hybrid Configuration (Backend Defaults + URL Overrides)**

Configuration follows a layered approach:

1. **Backend Configuration (Primary)**: Merchants configure default settings in Backoffice
   - Theme colors and branding
   - Business hours and booking rules
   - Default party size options
   - Custom messaging and terms

2. **URL Parameters (Optional Overrides)**: Merchants can override specific settings per page
   - `venueId` (required): Identifies the merchant venue
   - `theme` (optional): Override theme ('light', 'dark', 'auto')
   - `initialState` (optional): Widget display state ('minimized', 'open')
   - `other`

**Examples:**

Basic integration (all settings from backend):

```html
<script src="https://reservations.lightspeedhq.com/widget-loader.js?venueId=123"></script>
```

With URL overrides:

```html
<script src="https://reservations.lightspeedhq.com/widget-loader.js?venueId=123&theme=dark&initialState=open"></script>
```

Advanced: Using JavaScript object for complex configuration:

```html
<script>
  window.lsk_reservation = {
    venueId: 123,
    theme: "dark",
    initialState: "open",
    onBookingComplete: function (booking) {
      console.log("Booking completed:", booking);
    },
  };
</script>
<script src="https://reservations.lightspeedhq.com/widget-loader.js"></script>
```

**Why Hybrid Approach:**

- **Easy for most merchants**: Just copy-paste script tag, all settings managed in Backoffice
- **Flexible for advanced use cases**: Can override per page (e.g., special events page)
- **Best of both worlds**: Combines simplicity of backend config with power of URL parameters

### Configuration Resolution Flow

```mermaid
flowchart TD
    Start([Widget loads]) --> ParseURL[Parse URL parameters]
    ParseURL --> CheckWindow{window.lsk_reservation<br/>exists?}

    CheckWindow -->|Yes| MergeWindow[Merge window config]
    CheckWindow -->|No| FetchBackend[Fetch Backoffice config]
    MergeWindow --> FetchBackend

    FetchBackend --> BackendConfig[Backend Configuration:<br/>- Theme colors<br/>- Business hours<br/>- Booking rules<br/>- Default settings]

    BackendConfig --> ApplyDefaults[Apply backend defaults]
    ApplyDefaults --> CheckOverrides{URL params<br/>provided?}

    CheckOverrides -->|Yes| Override[Override specific settings:<br/>- theme<br/>- initialState<br/>- partySize<br/>- date]
    CheckOverrides -->|No| UseFinal

    Override --> UseFinal[Final configuration]
    UseFinal --> InitWidget[Initialize widget with config]
    InitWidget --> End([Widget ready])

    style BackendConfig fill:#e3f2fd
    style Override fill:#fff9c4
    style UseFinal fill:#c8e6c9
```

**Configuration Priority (Highest to Lowest):**

1. **URL Parameters** - Overrides everything (e.g., `?theme=dark`)
2. **JavaScript Window Object** - Custom configuration via `window.lsk_reservation`
3. **Backoffice Settings** - Default configuration set by merchant
4. **System Defaults** - Fallback if nothing specified

### Access Control and Licensing

**Addon Requirement**

The reservation widget is only available to merchants who have purchased the reservation addon:

- **License Verification**: Widget loader checks if merchant has active reservation addon
- **Backend Validation**: `lsk-reservation-service` validates addon status via `activation-manager`
- **Graceful Degradation**: If addon not active, widget shows upgrade message with link to purchase

**License Check Flow:**

1. Widget page loads with `venueId` parameter
2. Backend checks addon status for business location
3. If addon active → Load widget normally
4. If addon inactive or expired → Display warning message

### Security

**Domain Validation**

To prevent unauthorized widget usage on non-merchant domains:

1. **Allowed Domains List**: Merchants configure allowed domains in Backoffice (e.g., `example.com`, `www.example.com`)
2. **Backend Validation**: Widget page checks `Referer` header against allowed domains
3. **Fallback**: If validation fails, display error message: "Widget not authorized for this domain"

**Ad Blocker Compatibility**

- **URL Naming**: Use generic, non-tracking-like paths to avoid ad blocker filters
- **Detection**: Loader script includes fallback detection to display message if blocked
- **Graceful Degradation**: If widget fails to load, show direct link to reservation page

**Data Security**

- **PII Handling**: Widget handles guest personal information (name, email, phone)
- **HTTPS Only**: All widget traffic over encrypted connections
- **CORS Policy**: Strict CORS headers prevent unauthorized API access
- **No Sensitive Data in URL**: Guest information sent via secure POST requests, never URL parameters
- **Session Management**: Short-lived session tokens for booking flow

### Analytics

**Widget Usage Tracking**

The system will track key metrics to measure widget effectiveness:

**Implementation Methods:**

1. **Client-Side Detection**: Widget JavaScript detects iframe context

```javascript
const isEmbedded = window.self !== window.top;
// Send analytics event with context
```

2. **Server-Side Tracking**: Backend analyzes HTTP `Referer` header
   - Widget embedded: Referer = merchant domain
   - Direct access: Referer = lightspeed domain or empty

**Metrics to Track:**

- **Widget Loads**: Number of times widget is loaded on merchant sites
- **Conversion Rate**: Bookings completed via widget vs. abandoned
- **Time to Book**: Duration from widget open to booking completion
- **Error Rate**: Failed bookings and reasons
- **Device Distribution**: Desktop vs. mobile usage
- **Merchant Adoption**: Number of merchants installing widget
- **Domain Distribution**: Which merchant domains use the widget most

**Integration:**

- Events sent to existing analytics infrastructure (e.g., Amplitude, Mixpanel)
- Dashboards created for product team to monitor performance
- A/B testing capability for widget variations

## Dependencies

**Internal Systems:**

- **lsk-reservation-service** (RFC 0070): Primary backend for availability checks and booking creation
  - API endpoints: `/available-timeslots`, `/reservations` (POST)
  - Real-time availability calculation
  - Booking confirmation and email notifications

- **activation-manager**: Addon license verification service
  - Validates if merchant has active reservation addon
  - Returns addon status (active/inactive/expired)
  - Widget checks addon status before loading booking functionality
  - Addon status cached for 1 hour to minimize load

- **Reservation Website**: Widget-specific pages served from existing reservation frontend
  - New route: `/reservation/{venue-id}/widget`
  - Reuses existing React components and styling system
  - Responsive layout optimized for iframe embedding

- **CDN (CloudFront + S3)**: Hosts loader script and static assets
  - Global distribution for low latency
  - Versioned assets for cache control
  - Fallback to primary region if CDN fails

- **Backoffice**: Configuration interface for merchants
  - Widget settings page (theme, behavior, allowed domains)
  - Installation instructions and code snippets
  - Analytics dashboard
  - Addon subscription management

**External Systems:**

- None directly, but widget operates within merchant websites (external to our infrastructure)

## Alternatives Considered / Prior Art

### Current Solution: Direct Links

**Current Approach:**

- Merchants link to `lightspeed.app/reservation/{venue-id}`
- Users leave merchant website to complete booking
- High drop-off rates due to context switching

**Why Not Sufficient:**

- Breaks user flow and trust
- No customization or branding control
- Competitors offer better embedded experiences

### Alternative 1: Direct Widget Injection (No Iframe)

**Description:**
Loader script downloads widget JavaScript and injects directly into merchant page (similar to chat widgets like Intercom).

**Pros:**

- More flexible UI integration
- Better performance (no iframe overhead)
- Easier communication with merchant page

**Cons:**

- **Security Risk**: Widget JavaScript runs in merchant domain (XSS vulnerabilities)
- **CSS Conflicts**: Merchant styles can break widget appearance
- **Complex Isolation**: Requires Shadow DOM or strict CSS namespacing
- **Longer Development Time**: Need robust isolation mechanisms

**Decision:** Rejected due to security and complexity concerns.

### Alternative 2: Iframe Pointing to Existing Reservation Page

**Description:**
Iframe points to current reservation page (`/reservation/{venue-id}/reservation`) without modifications.

**Pros:**

- Zero development effort
- Reuses everything that exists today
- Immediate availability

**Cons:**

- **Poor UX**: Existing page not optimized for iframe embedding
- **Navigation Issues**: Full-page navigation doesn't work well in iframe
- **No Widget-Specific Features**: Can't add widget-specific customization
- **Mobile Experience**: Not optimized for small embedded contexts

**Decision:** Rejected - minimal effort but poor user experience.

### Alternative 3: Third-Party Widget Solutions

**Description:**
Continue relying on OpenTable, Resy, or other third-party reservation widgets.

**Pros:**

- No development required
- Proven solutions

**Cons:**

- **Lost Revenue**: Merchants pay competitors instead of us
- **Data Loss**: Reservation data lives in competitor systems
- **No Integration**: Can't integrate with our POS or other products
- **Limited Control**: Can't customize or improve widget

**Decision:** This is the problem we're solving, not a viable alternative.

### Chosen Solution: Iframe with New Widget-Specific Page

Balances security (iframe isolation), development speed (reuse existing infrastructure), and user experience (widget-optimized UI).

## Operations

**Team Ownership:**

The team responsible for reservations will own and maintain the widget. Note: There is currently no clearly defined team structure, and the team responsible may change in the future.

**Regular Processes:**

1. **Merchant Support**
   - **Installation Assistance**: Help merchants install and configure widget
   - **Troubleshooting**: Debug issues with widget display or functionality
   - **Training Materials**: Maintain documentation and video guides
   - **Expected Volume**: Low initially (beta program), scaling with adoption

2. **Monitoring and Maintenance**
   - **Uptime Monitoring**: Alert on widget availability issues
   - **Performance Tracking**: Monitor load times and error rates
   - **Analytics Review**: Weekly review of conversion metrics
   - **Browser Compatibility**: Test new browser versions quarterly

3. **Feature Requests and Improvements**
   - **Merchant Feedback**: Collect and prioritize widget enhancement requests
   - **Competitive Analysis**: Monitor competitor widget features
   - **A/B Testing**: Run experiments on widget UI and flow

**Operational Impact on Other Teams:**

- **Infrastructure Team**: Initial CDN setup and monitoring (one-time)
- **Security Team**: Review before launch (one-time), periodic security audits
- **Customer Support**: Trained on widget installation troubleshooting
- **Marketing Team**: Create merchant communication materials

**Monitoring and Alerts:**

- Widget load success rate (alert if < 99%)
- Booking completion rate via widget (alert if drops > 10%)
- API response times for widget endpoints (alert if p95 > 2s)
- CDN health and failover status

## Risks

### Technical Risks

**1. Performance / Latency**

**Risk:** Widget loading or API calls could slow down merchant websites, leading to merchant complaints and removal.

**Impact:** High - Slow widgets damage merchant experience and harm adoption

**Mitigation:**

- Implement aggressive caching for availability lookups (5-minute TTL)
- Use global CDN (CloudFront) for widget assets (< 100ms load time target)
- Lazy-load widget iframe (only when user scrolls into view)
- Async script loading (non-blocking)
- Performance budget: Widget < 50KB gzipped, load time < 2s on 3G

**2. Browser and Device Compatibility**

**Risk:** Widget may not work correctly across different browsers, devices, or CMS platforms (WordPress, Shopify, Wix).

**Impact:** Medium - Limits merchant adoption if widget breaks on popular platforms

**Mitigation:**

- Browser support: Modern browsers (Chrome, Firefox, Safari, Edge - last 2 versions)
- Responsive design: Mobile-first approach
- Testing matrix: Test on top 5 CMS platforms before launch
- Graceful degradation: Show direct link if widget fails to load
- Extensive QA across devices and browsers during beta phase

**3. Ad Blocker Interference**

**Risk:** Ad blockers may prevent widget from loading (blocking tracking scripts).

**Impact:** Medium - Reduced functionality for users with ad blockers

**Mitigation:**

- Use generic, non-tracking-like file names (`widget-loader.js`, not `analytics.js`)
- Detection script displays message: "Please disable ad blocker to book reservation"
- Fallback to direct reservation link
- Monitor blocked load rate in analytics

### Security and Privacy Risks

**4. Data Privacy and PII Handling**

**Risk:** Widget handles guest personal information (name, email, phone). Data breach or mishandling could violate GDPR/CCPA.

**Impact:** Critical - Legal liability, reputational damage

**Mitigation:**

- HTTPS only for all widget traffic
- Strict CORS policies (allowed domains only)
- No PII in URL parameters (POST requests only)
- Security review by Security Team before launch
- GDPR/CCPA compliance review
- Short-lived session tokens (30-minute expiry)

**5. Payment Data Handling**

**Risk:** If widget supports payment deposits in the future, PCI DSS compliance required.

**Impact:** Critical - Legal and financial liability

**Mitigation:**

- **Phase 1 (MVP)**: No payment handling in widget
- **Future**: Use tokenization only (Stripe Elements or similar), never handle raw card data
- PCI DSS compliance review before payment features launched

**6. Unauthorized Domain Usage**

**Risk:** Widget could be embedded on unauthorized domains (competitors, malicious sites).

**Impact:** Medium - Brand damage, potential abuse

**Mitigation:**

- Domain allowlist configured in Backoffice
- Backend validates `Referer` header
- Display error message on unauthorized domains
- Regular monitoring for unauthorized usage

**7. Addon License Bypass Attempts**

**Risk:** Merchants without active reservation addon might attempt to use widget without payment.

**Impact:** Low - Revenue loss if bypass is possible

**Mitigation:**

- Addon status checked on every widget load (with 1-hour cache)
- Backend validation in `lsk-reservation-service` prevents API usage without addon
- Widget displays upgrade prompt if addon inactive/expired
- All booking API endpoints validate addon status server-side
- Cannot be bypassed by manipulating client-side code
- Analytics track addon check failures for monitoring

### Operational Risks

**8. Team Ownership Uncertainty**

**Risk:** No clearly defined team structure. Future team changes could impact maintenance.

**Impact:** Medium - Support and maintenance could suffer

**Mitigation:**

- Document architecture and operational procedures thoroughly
- Create runbooks for common issues
- Ensure knowledge transfer if team changes
- Assign interim ownership to current reservation team leads

**9. Merchant Support Volume**

**Risk:** High support volume for widget installation and troubleshooting could overwhelm support teams.

**Impact:** Medium - Poor merchant experience, slow adoption

**Mitigation:**

- Comprehensive self-service documentation
- Video tutorials for installation
- Beta program to identify common issues before general release
- Train customer support team before launch
- Auto-diagnostic tools in Backoffice (check if widget is installed correctly)

### Business Risks

**10. Low Merchant Adoption**

**Risk:** Merchants may not adopt widget if installation is complex or value proposition is unclear.

**Impact:** High - Project failure if adoption is low

**Mitigation:**

- Beta program with pilot merchants to validate value
- Simple copy-paste installation (single script tag)
- Marketing campaign highlighting benefits
- Success metrics dashboard for merchants (show increased bookings)
- Regular merchant feedback and iteration

**11. Competitive Feature Parity**

**Risk:** Competitors (OpenTable, Resy) have mature widgets with more features.

**Impact:** Medium - May lose competitive advantage

**Mitigation:**

- MVP focuses on core booking flow (fast time to market)
- Iterative improvement based on merchant feedback
- Roadmap for advanced features (table selection, special requests, etc.)
- Leverage existing integration with our POS as differentiator
