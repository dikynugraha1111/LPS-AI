# M10 — Customer Dashboard & Monitoring: User Stories

## US-CD-01: Customer Portal Dashboard
**As a** Customer,  
**I want to** see a summary dashboard when I log in,  
**So that** I can quickly see the status of my active nominations, voyages, and current weather conditions.

**Acceptance Criteria:**
- [ ] After login, customer is redirected to /customer/dashboard
- [ ] Dashboard shows a summary of active nominations (status not Draft or Payment Confirmed) with status badges
- [ ] Dashboard shows active voyages (Payment Confirmed) with vessel name and last known position
- [ ] Dashboard shows a weather widget with current conditions and a color-coded danger level (green=Normal, yellow=Warning, red=Critical)
- [ ] "New Nomination" quick action button is prominently visible
- [ ] Dashboard data refreshes automatically every 60 seconds without full page reload
- [ ] If there are no active nominations, dashboard shows "Belum ada nominasi aktif. Buat nominasi baru." with a link

## US-CD-02: View All Nominations with Status
**As a** Customer,  
**I want to** see a full list of all my nominations and their current status,  
**So that** I can track which ones need my attention and access their details.

**Acceptance Criteria:**
- [ ] Nomination list page shows all nominations: Draft, In Progress, and Completed
- [ ] Each row shows: Nomination Number (or "Draft"), Vessel Name, ETA, Status badge, Created date
- [ ] Status badges are color-coded: Draft (gray), Pending/Submitted (blue), Approved (green), Need Revision (orange), Payment-related (purple), Confirmed (dark green), Rejected/Failed (red)
- [ ] Filter tabs allow viewing: All | Active (in-progress) | Draft | Completed
- [ ] Clicking a row navigates to the nomination detail page
- [ ] "New Nomination" button is visible at the top of the page

## US-CD-03: Track Active Voyage Position
**As a** Customer,  
**I want to** see the real-time position of my vessel on a map after payment is confirmed,  
**So that** I can monitor my vessel's status during the STS operation.

**Acceptance Criteria:**
- [ ] Voyage tracking page is only accessible for nominations with status PAYMENT_CONFIRMED
- [ ] A map displays the last known AIS position of the vessel
- [ ] The assigned anchor point is shown on the map as a marked zone
- [ ] Vessel information panel shows: Vessel Name, Anchor Point, ETB, last AIS update timestamp
- [ ] If AIS position is older than 10 minutes, a warning is displayed: "Data posisi mungkin tidak terkini"
- [ ] Map and data are read-only (no interaction beyond zoom/pan)

## US-CD-05: Document Master — Upload and Reuse Documents
**As a** Customer,  
**I want to** have a personal document library where I can store my company documents once and reuse them across multiple nominations,  
**So that** I don't need to re-upload the same file every time I submit a nomination.

**Acceptance Criteria:**
- [ ] "Document Master" menu item is visible in the customer sidebar navigation
- [ ] Document Master page shows a list/table of all uploaded documents with: File Name, Type (badge), Size, Uploaded At
- [ ] Customer can upload a new document via file picker (PDF/JPG/PNG, max 10MB)
- [ ] On successful upload, a success notification is shown and the document appears immediately in the list
- [ ] No "Delete" or "Edit" buttons are visible anywhere on the Document Master page
- [ ] Customer can download or preview each document (read-only)
- [ ] Customer cannot access another customer's documents
- [ ] API-level DELETE/PATCH requests to documents return 403 Forbidden

## US-CD-06: Select Document from Master in Nomination Form
**As a** Customer,  
**I want to** choose existing documents from my Document Master when filling a nomination form,  
**So that** I can reuse documents I've already uploaded without selecting the file again.

**Acceptance Criteria:**
- [ ] Each document upload section in the nomination form shows two options: "Upload Baru" and "Pilih dari Document Master"
- [ ] Selecting "Upload Baru" shows the standard file picker; on upload, file is saved to Document Master AND attached to the nomination
- [ ] Selecting "Pilih dari Document Master" opens a modal showing the customer's document list with search/filter
- [ ] Customer can select one document from the modal; selected document is attached to the nomination (no file duplication)
- [ ] Selected document name is shown in the upload card with a confirmation indicator
- [ ] Both upload methods satisfy the document requirement for nomination submission

## US-CD-04: View Weather Conditions and Alert History
**As a** Customer,  
**I want to** see the current weather and recent alerts for the STS Bunati area,  
**So that** I can plan my vessel's arrival and stay informed about operational conditions.

**Acceptance Criteria:**
- [ ] Weather page shows current conditions: wave height (m), wind speed (km/h), visibility (km), danger level
- [ ] Danger level is color-coded: Normal (green), Warning (yellow, wave > 2.5m), Critical (red, wave > 3m)
- [ ] Alert history table shows alerts from the last 24 hours with: level, message, time, resolved status
- [ ] Ongoing alerts are highlighted; resolved alerts are shown with resolved timestamp
- [ ] Weather data is read-only (customer cannot broadcast or create alerts)
- [ ] Page shows "Last updated" timestamp for weather data
