# Activity Diagram: AD-UC-1.05 Upload External Maintenance & Repair Invoices

| Field | Value |
| :---- | :---- |
| AD ID | AD-UC-1.05 |
| Use case | UC-1.05 Upload External Maintenance & Repair Invoices |
| Feature | F-01 Vehicle Identification & Status |
| Primary actor | Car Owner |
| Lanes | Car Owner · TASCO Mobility System · Electronic Invoice Portal |
| Inputs (L3-L5) | Context Diagram v1 · Use Case Diagram v1 · Sequence: SD-UC-1.05 (Figure 9\) · W1 rows (FR-01-05-FE, FR-01-05-BE, FR-01-06-BE, FR-01-07-INT, FR-01-07-BE) · Screens: S-F01-05 (10 screens overview) |
| Trigger | Car owner selects upload external maintenance and repair invoice |
| Preconditions | Vehicle profile exists in the system and car owner is authenticated |
| Success end state | `stop1` (via `s6`): External invoice file is successfully validated, parsed (or manually completed by the car owner), cross-checked, and merged into the vehicle maintenance history timeline |
| Failure end state(s) | `stop1` (via `s1` or `s7`): invoice file is rejected due to invalid format/size, or the Electronic Invoice Portal returns an invalid status; an error message is shown, nothing is merged, and the car owner closes the popup. If the car owner chooses Chọn tệp khác instead (on screen 3 or on the Portal rejection popup, screen 6), the flow does not end: it loops back to `m1` and the upload starts again. `stop2`: the Portal API call fails on all 4 attempts (first attempt and 3 retries), the invoice is not merged and not rejected, and the car owner closes the Error API popup (the confirmed info stays on the screen) |
| FRs covered | FR-01-05-FE, FR-01-05-BE, FR-01-06-BE, FR-01-07-INT, FR-01-07-BE |
| Business rules applied | BR-FILE-01, BR-OCR-01, BR-INV-01, BR-RET-03 |
| Version / status | v0.6 · Reviewed |
| Author | Đinh Thu Thảo |

## 

1. ## **Diagram** 

```mermaid

swimlane-beta TB
  subgraph CAR["Car Owner"]
    a_0start@{ shape: sm-circ }
    a1("Open maintenance history and select upload invoice (FR-01-05-FE)")
    m1{" "}
    a2("Upload invoice file PDF, XML, or image (FR-01-05-FE)")
    a_edit("Review and edit extracted invoice info if needed, then confirm (FR-01-05-FE)")
    d_choice{"Enter manually or choose another file?"}
    a_manual("Fill in missing fields or enter all invoice info manually, then confirm (FR-01-05-FE)")
    d_retry{"Retry or close?"}
    a_close("Close the Error API popup and stay on the screen with the confirmed info (FR-01-05-FE)")
    d_rej{"Close or choose another file?"}
    d_fe{"Choose another file or close?"}
    m4{" "}
    a3("See updated maintenance timeline or error message (FR-01-05-FE)")
    stop1@{ shape: fr-circ }
    stop2@{ shape: fr-circ }
  end
  subgraph SYS["TASCO Mobility System"]
    d1{"File format or size valid?"}
    s1("Reject invalid file and show error (FR-01-05-FE)")
    s2("Store uploaded file (FR-01-05-BE)")
    s3a("Detect real file type: XML, PDF, or image (FR-01-06-BE)")
    d_xml{"XML file?"}
    s3b("Parse XML tags directly, no OCR (FR-01-06-BE)")
    s3c("Extract PDF text or preprocess image and run OCR (FR-01-06-BE)")
    m2{" "}
    s3d("Extract and normalize tax code, invoice number, total amount, invoice date (FR-01-06-BE)")
    d2{"All 4 fields (tax code, invoice number, total amount, invoice date) extracted and valid?"}
    s_ocr("Notify OCR failure or missing fields and open manual entry screen, keep any extracted values (FR-01-06-BE)")
    s4("Display extracted data on edit and confirmation screen, highlight low-confidence fields (FR-01-06-BE)")
    m3{" "}
    s5("Send invoice info via API to verify with Electronic Invoice Portal (FR-01-07-INT)")
    d_api{"Did the Electronic Invoice Portal API respond successfully? (no timeout, response valid)"}
    s9("Wait 2 seconds, then retry (max 3 retries) (FR-01-07-INT)")
    s8("Max 3 retries exceeded: log error, notify API error and keep confirmed invoice info (FR-01-07-INT)")
    d3{"Invoice valid?"}
    s6("Merge validated external invoice into vehicle history (FR-01-07-BE)")
    s7("Reject invoice and show error message (FR-01-07-BE)")
  end
  subgraph PORTAL["Electronic Invoice Portal"]
    e1("Verify via API at Electronic Invoice Portal: check seller tax code exists, invoice details match, and status is not cancelled/replaced (FR-01-07-INT)")
  end

  a_0start --> a1 --> m1 --> a2 --> d1
  d1 -->|"[no]"| s1 --> d_fe
  d_fe -->|"[choose another file]"| m1
  d_fe -->|"[close]"| m4
  d1 -->|"[yes]"| s2 --> s3a --> d_xml
  d_xml -->|"[yes]"| s3b --> m2
  d_xml -->|"[no]"| s3c --> m2
  m2 --> s3d --> d2
  d2 -->|"[yes]"| s4 --> a_edit --> m3
  d2 -->|"[no]"| s_ocr --> d_choice
  d_choice -->|"[enter manually]"| a_manual --> m3
  d_choice -->|"[choose another file]"| m1
  m3 --> s5 --> e1 --> d_api
  d_api -->|"[success, response valid]"| d3
  d_api -->|"[fail/timeout & retries done < 3]"| s9 --> m3
  d_api -->|"[fail/timeout & 3 retries used]"| s8 --> d_retry
  d_retry -->|"[retry]"| m3
  d_retry -->|"[close]"| a_close --> stop2
  d3 -->|"[yes]"| s6 --> m4
  d3 -->|"[no]"| s7 --> d_rej
  d_rej -->|"[choose another file]"| m1
  d_rej -->|"[close]"| m4
  m4 --> a3 --> stop1
```

**Reading notes:**

- File Upload & Validation: The car owner uploads an external invoice file (PDF, XML, or image), and the system validates its format and size before storing it; if invalid, the file is rejected with an error (screen 3), and the car owner either chooses another file, which returns to the upload step via `m1`, or closes the modal (`d_fe`, ends at `stop1`).
- Extraction (XML vs OCR): The system detects the real file type. XML files are parsed directly from their tags (no OCR needed). PDF and image files go through text extraction or image preprocessing and OCR. In both cases the system extracts and normalizes four key fields: seller tax code, invoice number, total amount, and invoice date.
- Extraction Result: If all four fields are extracted in a valid format, the data is displayed on the edit and confirmation screen for the car owner to review, edit if needed, and confirm (low-confidence fields are highlighted). If the file is unreadable or any field is missing or invalid, the system notifies the car owner and opens the manual entry screen, keeping any values it did extract.
- Manual Entry or Re-upload: On the manual entry screen, the car owner chooses either to fill in the missing fields or enter all invoice info manually and confirm, or to choose another file, which returns to the upload step.
- Authentication Check: After the car owner confirms (from review or manual entry), the system sends the invoice information via API to the Electronic Invoice Portal. The Portal verifies the invoice and its response goes back to the TASCO Mobility System, which checks (`d_api`) whether the API responded successfully (no timeout, response valid). `d_api` is evaluated after every attempt, the same pattern as AD-UC-1.06.
- Automatic retry loop (BR-RET-03): `retries done` counts the retries already made, not the attempts. After attempt 1, 2 or 3 fails, `retries done` is 0, 1 or 2 (below 3), so the system waits 2 seconds (`s9`, max 3 retries) and loops back to merge node `m3`, which sends the same confirmed info again. This gives 1 initial request plus 3 retries, 4 attempts in total, the same limit as AD-UC-1.06. During the loop the invoice is neither accepted nor rejected. Each attempt times out after 5 seconds, so the worst case for one submission is about 26 seconds (4 attempts of 5 seconds plus 3 waits of 2 seconds).
- API Error after retries: If the 4th attempt also fails (`3 retries used`), the loop stops. The system logs the error, notifies the car owner with the Error API popup, and keeps the confirmed invoice info (`s8`); the invoice is not merged and not rejected. The popup offers two choices (`d_retry`): Thử lại sends the confirmed info again and starts a new cycle of up to 4 attempts via `m3`; Đóng closes the popup, the car owner stays on the screen with the confirmed info, and the flow ends at `stop2`.
- Result & Timeline Update: If the Portal responded and the invoice is valid, it is merged into the vehicle's maintenance history and the updated timeline is displayed; if the Portal responded with an invalid status (or the file was rejected at validation), an error message is shown (popup "Hóa đơn không tồn tại" with Đóng and Chọn tệp khác). An invalid status is a valid response, so it is never retried. After the rejection popup, the car owner either closes it (the flow ends at `stop1`) or chooses another file, which loops back to `m1` (`d_rej`) and the next upload starts again at `a2`, the same as `d_fe` and `d_choice`.
- Mermaid rendering note: the initial node is named `a_0start` on purpose. The swimlane layout finds loop-back edges with a depth-first search that visits nodes in alphabetical order of their ID, so the search must begin at the initial node. If it begins at another node (for example `a_close`), it reverses forward edges as well and the diagram is drawn upside down. When adding nodes, keep the initial node ID first in alphabetical order, and keep every retry or choose-another-file loop pointing back to a merge node (`m1`, `m3`).

2. ## **Node trace**

| Node | Kind | Text | Traces to |
| :---- | :---- | :---- | :---- |
| a1 | Action | Open maintenance history and select upload invoice | FR-01-05-FE |
| m1 | Merge | Merge path for initial upload and re-upload attempts | N/A (Control Node) |
| a2 | Action | Upload invoice file PDF, XML, or image | FR-01-05-FE |
| d1 | Decision | File format or size valid? | FR-01-05-FE / BR-FILE-01 |
| s1 | Action | Reject invalid file and show error | FR-01-05-FE / BR-FILE-01 |
| d_fe | Decision | Choose another file or close? | FR-01-05-FE / BR-FILE-01 |
| s2 | Action | Store uploaded file | FR-01-05-BE |
| s3a | Action | Detect real file type: XML, PDF, or image | FR-01-06-BE / BR-OCR-01 |
| d_xml | Decision | XML file? | FR-01-06-BE / BR-OCR-01 |
| s3b | Action | Parse XML tags directly, no OCR | FR-01-06-BE / BR-OCR-01 |
| s3c | Action | Extract PDF text or preprocess image and run OCR | FR-01-06-BE / BR-OCR-01 |
| m2 | Merge | Merge XML parsing path and PDF/image OCR path | N/A (Control Node) |
| s3d | Action | Extract and normalize tax code, invoice number, total amount, invoice date | FR-01-06-BE / BR-OCR-01 |
| d2 | Decision | All 4 fields (tax code, invoice number, total amount, invoice date) extracted and valid? | FR-01-06-BE / BR-OCR-01 |
| s_ocr | Action | Notify OCR failure or missing fields and open manual entry screen, keep any extracted values | FR-01-06-BE / BR-OCR-01 |
| s4 | Action | Display extracted data on edit and confirmation screen, highlight low-confidence fields | FR-01-06-BE / BR-OCR-01 |
| a_edit | Action | Review and edit extracted invoice info if needed, then confirm | FR-01-05-FE / BR-OCR-01 |
| d_choice | Decision | Enter manually or choose another file? | FR-01-05-FE / BR-OCR-01 |
| a_manual | Action | Fill in missing fields or enter all invoice info manually, then confirm | FR-01-05-FE / BR-OCR-01 |
| m3 | Merge | Merge confirmed invoice info from review and manual entry, the automatic retry path, and the retry path from the Error API popup | N/A (Control Node) |
| s5 | Action | Send invoice info via API to verify with Electronic Invoice Portal | FR-01-07-INT / BR-INV-01 |
| e1 | Action | Verify invoice and return authenticity result | FR-01-07-INT / BR-INV-01 |
| d_api | Decision | Did the Electronic Invoice Portal API respond successfully? (no timeout, response valid) | FR-01-07-INT / BR-INV-01, BR-RET-03 |
| s9 | Action | Wait 2 seconds, then retry (max 3 retries) | FR-01-07-INT / BR-RET-03 |
| s8 | Action | Max 3 retries exceeded: log error, notify API error and keep confirmed invoice info | FR-01-07-INT / BR-INV-01, BR-RET-03 |
| d_retry | Decision | Retry or close? | FR-01-07-INT / BR-RET-03 |
| a_close | Action | Close the Error API popup and stay on the screen with the confirmed info | FR-01-05-FE / BR-RET-03 |
| d3 | Decision | Invoice valid? | FR-01-07-BE / BR-INV-01 |
| s6 | Action | Merge validated external invoice into vehicle history | FR-01-07-BE / BR-INV-01 |
| s7 | Action | Reject invoice and show error message | FR-01-07-BE / BR-INV-01 |
| d_rej | Decision | Close or choose another file? | FR-01-05-FE / BR-INV-01 |
| m4 | Merge | Merge file rejected, invoice merged and rejection popup closed flows | N/A (Control Node) |
| a3 | Action | See updated maintenance timeline or error message | FR-01-05-FE |
| stop1 | Final | End: invoice merged, or file or invoice rejected with an error message | N/A (Control Node) |
| stop2 | Final | End: API error after max retries, popup closed, nothing merged | N/A (Control Node) |

## 

**Business rule description**

| Rule ID | Description | Applied Node |
| ----- | ----- | :---: |
| BR-FILE-01 | File Format and Size Rule: The system only accepts uploaded files in .pdf, .xml, or image formats (.jpg, .png), with a file size adhering to the system limit (e.g., <= 10MB). Invalid files are rejected immediately at the validation step. | d1 |
| BR-OCR-01 | Automated OCR Data Extraction & Review Rule: The system extracts four key invoice fields: seller tax code, invoice number, total amount, and invoice date. (1) File type is detected from the actual file content, not only the extension. (2) XML files are parsed directly from their tags and do not go through OCR. (3) PDF files with a text layer have their text extracted directly; scanned PDFs and images are preprocessed (rotation, deskew, noise removal, contrast enhancement) and then run through OCR, which returns text, position, and confidence per token. (4) Each field is located by keyword and position and then normalized and format-checked: **Tax code** is the seller's (not the buyer's), 10 digits or 13 digits (10 + 3-digit branch code), spaces and hyphens removed; **Invoice number** is digits only; **Total amount** is the final payable total (including VAT, not line-item or pre-tax amounts), thousand separators removed, converted to a positive integer in VND, and cross-checked against the amount in words when available; **Invoice date** is a valid date (dd/mm/yyyy), not in the future, and stored in ISO format. (5) If all four fields are extracted in a valid format, the data is shown on an edit and confirmation screen, so the car owner can review and edit if needed before confirming. (6) If the file is unreadable, or any field is missing or invalid, the file is not rejected: the system notifies the car owner, opens the manual entry screen with any extracted values kept, and the car owner either fills in the missing fields or enters all invoice info manually, or chooses another file to upload. (7) The car owner must provide all four fields before confirming. User edits and manual entries do not bypass BR-INV-01: the Portal still decides whether the invoice is valid. | d_xml, s3b, s3c, s3d, d2, s_ocr, s4, a_edit, d_choice, a_manual |
| BR-INV-01 | Invoice Authenticity Cross-Check Rule: After confirmation, invoice data is sent via API to the Electronic Invoice Portal for verification. An invoice is considered valid only if the tax code exists, details match, and the invoice status is not cancelled or replaced on the tax system. If the API call itself fails (no response, timeout, or server error), no validity result exists: the invoice is neither accepted nor rejected, it is not merged, the retry mechanism of BR-RET-03 applies, and once the automatic retries are used up the car owner is notified of the API error and can retry or close. If the Portal returns an invalid status, the invoice is rejected and not merged; the car owner can close the popup or choose another file to upload. | s5, e1, d_api, d3, s6, s7, s8, d_rej |
| BR-RET-03 | Invoice Portal API Retry & Fallback Rule: When the verification API call to the Electronic Invoice Portal fails (no response within the 5-second timeout of one attempt, server error, or an unreadable response), the system automatically retries up to a maximum of 3 retries (maximum 4 attempts in total: the first attempt and 3 retries), with a 2-second delay between consecutive attempts, sending the same confirmed invoice info each time. The worst case wait for one submission is about 26 seconds (4 attempts of 5 seconds plus 3 waits of 2 seconds). During the retries the invoice is neither accepted nor rejected. If all 4 attempts fail, the system stops retrying, logs the error, shows the Error API popup ("Không kiểm tra được hóa đơn") and keeps the confirmed invoice info. The popup offers two choices: **Thử lại** sends the confirmed info again and starts a new cycle of up to 4 attempts; **Đóng** closes the popup without changing the maintenance history, and the car owner stays on the screen with the confirmed info. A response in which the Portal says the invoice is invalid, fake or unmatched is a successful API response, not an API failure, so it is never retried and goes to the invalid branch (d3). | d_api, s9, s8, d_retry, a_close |

3. ## **Branches and acceptance tests**

| Branch | Given | When | Then |
| :---- | :---- | :---- | :---- |
| d1 [no] | Vehicle maintenance screen is open | Car owner uploads an unsupported file format (.exe, .txt) or oversized file | System rejects the file immediately and displays an error message with Chọn tệp khác |
| d_fe [choose another file] | Error validate FE screen is shown after the file was rejected | Car owner chooses Chọn tệp khác | System returns to the upload step via m1 so the car owner can upload a new file |
| d_fe [close] | Error validate FE screen is shown after the file was rejected | Car owner closes the modal | System closes the modal without changing the maintenance history and the flow ends at stop1 |
| d1 [yes] | Vehicle maintenance screen is open | Car owner uploads a valid PDF, XML, or image file under size limit | System accepts and stores the uploaded file, then detects the file type and starts extraction |
| d_xml [yes] | File is stored successfully | The uploaded file is an XML invoice | System parses the XML tags directly (no OCR) and extracts tax code, invoice number, total amount, and invoice date |
| d_xml [no] | File is stored successfully | The uploaded file is a PDF or image | System extracts PDF text or preprocesses the image and runs OCR, then extracts and normalizes the four fields |
| d2 [yes] | File is stored successfully | All four fields (tax code, invoice number, total amount, invoice date) are extracted in a valid format | System displays extracted data on the edit and confirmation screen (low-confidence fields highlighted) for the car owner to review, edit if needed, and confirm |
| d2 [no] | File is stored successfully | OCR or XML parsing fails (blurred scan, corrupt structure), or one or more fields are missing or invalid | System notifies the car owner, opens the manual entry screen with any extracted values kept, and offers the choice to enter manually or choose another file |
| d_choice [enter manually] | Manual entry screen is open after extraction failure or missing fields | Car owner fills in the missing fields or enters all invoice info manually and confirms | System accepts the confirmed info and sends it to the Electronic Invoice Portal for verification |
| d_choice [choose another file] | Manual entry screen is open after extraction failure or missing fields | Car owner chooses to select another file | System returns to the upload step so the car owner can upload a new file |
| d_api [success, response valid] | Car owner confirms invoice data and the system sends the API request (any attempt 1 to 4) | Electronic Invoice Portal is reachable and returns a valid response | System continues with the authenticity check and evaluates the returned status (d3) |
| d_api [fail/timeout & retries done < 3] | Car owner confirms invoice data and the system sends the API request (attempt 1, 2 or 3) | The API call fails (no response within 5 seconds, or server error) | System does not merge or reject the invoice, waits 2 seconds, then loops back via merge node m3 and sends the same confirmed info again (max 3 retries) |
| d_api [fail/timeout & 3 retries used] | The API call failed on attempts 1, 2 and 3, and 3 retries have been made | The API call also fails on the 4th attempt | System stops retrying, logs the error, does not merge or reject the invoice, notifies the car owner with the Error API popup, keeps the confirmed invoice info, and offers Thử lại or Đóng |
| d_retry [retry] | Error API popup is shown after all 4 attempts failed | Car owner chooses Thử lại | System sends the confirmed invoice info to the Electronic Invoice Portal again and starts a new cycle of up to 4 attempts |
| d_retry [close] | Error API popup is shown after all 4 attempts failed | Car owner chooses Đóng | System closes the popup without changing the maintenance history; the car owner stays on the screen with the confirmed info, and the flow ends at stop2 |
| d3 [yes] | Car owner confirms invoice data and the API call to the Portal succeeded | Electronic Invoice Portal confirms invoice authenticity and returns valid status | System merges the validated external invoice into the vehicle maintenance history and displays the updated timeline |
| d3 [no] | Car owner confirms invoice data and the API call to the Portal succeeded | Electronic Invoice Portal returns invalid, fake, or unmatched status | System rejects the invoice without retrying, prevents merging, and displays the error popup with Đóng and Chọn tệp khác |
| d_rej [choose another file] | Error validate BE popup is shown after the Portal returned an invalid status | Car owner chooses Chọn tệp khác | System closes the popup and returns the car owner to the upload step via m1 so the car owner can upload a new file; the next upload starts again at a2 |
| d_rej [close] | Error validate BE popup is shown after the Portal returned an invalid status | Car owner chooses Đóng | System closes the popup without changing the maintenance history and the flow ends at stop1 |

4. ## **Object states used** ## 

| Object | States in this diagram | Full state list (all diagrams) |
| :---- | :---- | :---- |
| Invoice File | Uploaded -> Stored -> Parsed -> Verified / Rejected (stays Parsed when the API call to the Portal fails, during the retries and after the retries are used up) | Uploaded, Stored, Parsed, Verified, Archived, Rejected |
| Maintenance Timeline | Consolidated -> Pending Update -> Updated | Initial, Pending Update, Updated, Synced |

5. ## **Screens touched**

| Actor action | Screen ID |
| :---- | :---- |
| Open maintenance history and select upload invoice (a1) | S-F01-02 -> S-F01-05 (Upload invoice modal, screen 1 Default) |
| Upload invoice file PDF, XML, or image (a2) | S-F01-05 (Upload invoice modal, screen 2 Loading) |
| Reject invalid file and show error (s1) | S-F01-05 (Error validate FE, screen 3) |
| Choose another file or close after file rejection (d_fe) | S-F01-05 (Error validate FE, screen 3): Chọn tệp khác goes to S-F01-05 (Upload invoice modal); closing the modal goes to S-F01-02 |
| Display extracted data on edit and confirmation screen, highlight low-confidence fields (s4) | S-F01-05 (Review, edit and confirmation, screen 5) |
| Review and edit extracted invoice info if needed, then confirm (a_edit) | S-F01-05 (Review, edit and confirmation, screen 5) |
| Notify OCR failure or missing fields and open manual entry screen, keep any extracted values (s_ocr) | S-F01-05 (OCR incomplete, screen 4) |
| Choose to enter manually or choose another file (d_choice) | S-F01-05 (OCR incomplete, screen 4) -> S-F01-05 (Upload invoice modal) if "choose another file" |
| Fill in missing fields or enter all invoice info manually, then confirm (a_manual) | S-F01-05 (OCR incomplete, screen 4) |
| Reject invoice and show error message (s7) | S-F01-05 (Error validate BE popup, screen 6) |
| Wait 2 seconds, then retry (s9) | S-F01-05 (Review or OCR incomplete screen in a processing state, to be added: see open question 6) |
| Max 3 retries exceeded: log error, notify API error and keep confirmed invoice info (s8) | S-F01-05 (Error API popup, screen 7) |
| Choose to close or choose another file after rejection (d_rej) | S-F01-05 (Error validate BE popup, screen 6): Đóng closes the popup; Chọn tệp khác goes to S-F01-05 (Upload invoice modal) |
| Choose to retry or close (d_retry) | S-F01-05 (Error API popup, screen 7): Thử lại sends the confirmed info again and starts a new cycle (with a processing state: see open question 6); Đóng goes to a_close |
| Close the Error API popup and stay on the screen with the confirmed info (a_close) | S-F01-05 (Review or OCR incomplete screen behind the popup, screen 5 or 4) |
| See updated maintenance timeline or error message (a3) | S-F01-05 (Success popup, screen 8) -> S-F01-02 on success; S-F01-05 (screen 3 or 6) on rejection |

**Screen state note:** The OCR incomplete / manual entry state is shared by every case where extraction is incomplete (1 to 4 fields missing or invalid, or an unreadable file). Extracted values are pre-filled and missing fields are flagged; the confirm button stays disabled until all 4 fields are provided. The number of pre-filled fields varies with the extraction result, so no separate screen is needed per case. The Error API state is one popup shared by every case where the API call to the Electronic Invoice Portal fails; it shows Thử lại and Đóng after the retries are used up. The screen behind it keeps the confirmed info.
## 

6. ## **Open questions** ## 

| \# | Question | Blocking? | Default applied |
| :---- | :---- | :---- | :---- |
| 1 | Maximum allowable upload file size for images and PDFs? | No | Set limit to 10 MB per invoice file |
| 2 | Confidence threshold below which an extracted field is highlighted for review? | No | Set threshold to 80% |
| 3 | Does the Electronic Invoice Portal API require additional fields beyond tax code, invoice number, total amount, and invoice date (e.g., invoice series/symbol)? | Yes (for FR-01-07-INT) | Send the four extracted fields only; confirm with the integration team |
| 4 | Which OCR engine is used (self-hosted or external service)? This affects invoice data security. | No | Not specified; the diagram stays engine-agnostic |
| 5 | What are the timeout and retry rules for the API call to the Electronic Invoice Portal (how many automatic retries before the API error is shown)? | No | Confirmed by the author: BR-RET-03, 3 automatic retries after the first attempt (4 attempts in total), 2 seconds apart, same as BR-RET-02 in AD-UC-1.06; Thử lại in the popup starts a new cycle; confirm with the integration team |
| 6 | The car owner waits on the screen during the attempts, but the 10 screens have no processing state for the verification step. | No (for the diagram), needed before build | Add one, e.g. the confirm button or Thử lại shows "Đang xử lý..." and is disabled, like screen 2, so the car owner does not press twice; UX to design |
| 7 | What is the timeout of one API attempt before it counts as failed? | No | Set to 5 seconds per attempt (BR-RET-03), so the worst case for one submission is about 26 seconds; engineering to confirm against the measured response time of the Portal (suggested timeout: about 2 times its p95, kept between 3 and 5 seconds) |
| 8 | Should the number of Thử lại presses be capped across cycles? | No | No cap: each press is a deliberate action; use a server-side rate limit if the Portal needs protection |
| 9 | Is repeating the verification call free of side effects on the Portal side? | No | Assumed read-only, so safe to repeat; confirm with the integration team |
| 10 | Should Chọn tệp khác after a Portal rejection (screen 6) be a loop-back to m1? | No | Yes: drawn as a loop-back to m1, like d_fe and d_choice. The initial node is named a_0start so that the Mermaid layout finds the loop-back edges correctly (see the Mermaid rendering note) |

## 

7. ## **Verification log** ## 

| Check | Result | By | Date |
| :---- | :---- | :---- | :---- |
| Renders on mermaid.live | Pass | Đinh Thu Thảo | 2026-10-26 |
| Every decision has guards on all outgoing flows, no overlap, no gap | Pass | Đinh Thu Thảo | 2026-10-26 |
| No action has two incoming flows (merge used instead) | Pass | Đinh Thu Thảo | 2026-10-26 |
| Every fork has a matching join | Pass | Đinh Thu Thảo | 2026-10-26 |
| Every system action cites an FR | Pass | Đinh Thu Thảo | 2026-10-26 |
| Every FR of this use case appears in the diagram | Pass | Đinh Thu Thảo | 2026-10-26 |
| Lane names match the Context Diagram and the use case actors; no Web App, API or database lanes | Pass | Đinh Thu Thảo | 2026-10-26 |
| Related sequence diagram uses the same names and FR IDs; its alt fragments appear here as decisions | Pass | Đinh Thu Thảo | 2026-10-26 |
| AI critic prompt run, findings resolved or logged as open questions | Pass | Đinh Thu Thảo | 2026-10-26 |
