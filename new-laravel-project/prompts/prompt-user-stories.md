Look at the file `@docs/project-description.md` and work out user stories for the project.

Ask me any clarifying questions using AskUserQuestion tool, to improve the result.

Save result in the file `@docs/user-stories.md`.

Below you will see an example Project Description and example User Stories from it. Please mimic similar structure and depth of details.

---

# Example Project Description

I'm building VisaNauta, a marketplace platform that connects
immigration clients with licensed Regulated Canadian Immigration
Consultants (RCICs) in Canada. Think "Upwork for immigration
consultants."

I need an experienced full-stack developer to build the MVP
(Minimum Viable Product) from scratch.

WHAT I NEED BUILT:

This is a marketplace platform with 3 user types:

1. Clients (people seeking immigration services)
2. RCICs (immigration consultants)
3. Admin (me)

CORE FEATURES:

✓ User Authentication & Profiles
  - Sign up / Login (email + Google OAuth)
  - Separate dashboards for Clients, RCICs, and Admin
  - Profile management (photo, bio, credentials)

✓ RCIC Directory & Search
  - Browse all registered RCICs
  - Filter by: specialty, language, location, price, rating
  - Sort by: rating, price, experience, availability
  - RCIC profile pages with detailed information

✓ Booking System
  - Calendar showing RCIC availability
  - Time slot selection
  - Consultation type selection (video/phone/in-person)
  - Booking confirmation with details

✓ Payment Processing
  - Stripe integration (I already have Stripe account)
  - Secure payment for consultations
  - Automatic commission split (platform takes 20%, RCIC gets 80%)
  - Payment confirmation emails

✓ Review & Rating System
  - 5-star rating system
  - Text reviews from clients
  - Display average rating on RCIC profiles
  - Sort RCICs by rating

✓ Email Notifications
  - Booking confirmations (client + RCIC)
  - Reminders 24 hours before consultation
  - "Rate your experience" emails after consultation
  - Use SendGrid or similar (I can provide API key)

✓ Admin Dashboard
  - View all users, RCICs, bookings
  - Approve/reject RCIC applications
  - Manage platform settings
  - View revenue analytics

---

# Example User Stories

## Overview

This document contains user stories for VisaNauta, a marketplace platform connecting immigration clients with Regulated Canadian Immigration Consultants (RCICs).

**User Types:**
- **Client** - People seeking immigration services
- **RCIC** - Regulated Canadian Immigration Consultants
- **Admin** - Platform administrator
- **Visitor** - Unauthenticated user browsing the platform

---

## 1. Authentication & Registration

### US-1.1: Client Email Registration
**As a** Client
**I want to** register with my email and password
**So that** I can create an account to book consultations

**Acceptance Criteria:**
- [ ] Registration form collects: name, email, password, password confirmation
- [ ] Email must be unique in the system
- [ ] Password must meet minimum security requirements (8+ characters)
- [ ] User receives email verification link
- [ ] Account is created with "client" role
- [ ] User is redirected to client dashboard after verification

**Expected Result:** Client account is created and user can access client-specific features.

---

### US-1.2: Client Google OAuth Registration
**As a** Client
**I want to** register using my Google account
**So that** I can quickly create an account without remembering another password

**Acceptance Criteria:**
- [ ] "Sign up with Google" button is visible on registration page
- [ ] Clicking initiates Google OAuth flow
- [ ] If email already exists, account is linked to Google
- [ ] If new email, new client account is created
- [ ] User is redirected to client dashboard

**Expected Result:** Client can authenticate via Google and access their dashboard.

---

### US-1.3: RCIC Registration & Application
**As an** RCIC
**I want to** register and apply to join the platform
**So that** I can offer my immigration consulting services

**Acceptance Criteria:**
- [ ] Registration form collects: name, email, password, RCIC license number
- [ ] RCIC must provide proof of credentials (upload document)
- [ ] Account is created with "rcic" role and "pending" status
- [ ] RCIC receives confirmation email that application is under review
- [ ] Admin receives notification of new RCIC application
- [ ] RCIC cannot access booking features until approved

**Expected Result:** RCIC application is submitted and awaiting admin approval.

---

### US-1.4: User Login
**As a** registered user (Client, RCIC, or Admin)
**I want to** log in to my account
**So that** I can access my personalized dashboard

**Acceptance Criteria:**
- [ ] Login form accepts email and password
- [ ] "Login with Google" option available
- [ ] Invalid credentials show appropriate error message
- [ ] Successful login redirects to role-specific dashboard
- [ ] "Remember me" option persists session

**Expected Result:** User is authenticated and redirected to their appropriate dashboard.

---

### US-1.5: Password Reset
**As a** registered user
**I want to** reset my password if I forget it
**So that** I can regain access to my account

**Acceptance Criteria:**
- [ ] "Forgot password" link on login page
- [ ] User enters email address
- [ ] Password reset link sent to email (valid for 60 minutes)
- [ ] User can set new password via reset link
- [ ] Confirmation message shown after successful reset

**Expected Result:** User receives reset email and can set a new password.

---

## 2. Profile Management

### US-2.1: Client Profile Management
**As a** Client
**I want to** manage my profile information
**So that** RCICs can know who they're consulting with

**Acceptance Criteria:**
- [ ] Client can upload/change profile photo
- [ ] Client can edit: name, phone number, bio
- [ ] Client can add immigration goals/interests (optional)
- [ ] Changes are saved and reflected immediately
- [ ] Profile photo has size/format restrictions (max 2MB, jpg/png)

**Expected Result:** Client profile is updated and visible to RCICs during bookings.

---

### US-2.2: RCIC Profile Management
**As an** RCIC
**I want to** manage my professional profile
**So that** clients can learn about my expertise and decide to book with me

**Acceptance Criteria:**
- [ ] RCIC can upload/change profile photo
- [ ] RCIC can edit: name, bio, phone number
- [ ] RCIC can set specialties (multi-select from predefined list):
  - Express Entry
  - Family Sponsorship
  - Work Permits
  - Study Permits
  - Provincial Nominee Programs
  - Citizenship
  - Refugee/Asylum
  - Business Immigration
- [ ] RCIC can set languages spoken (multi-select)
- [ ] RCIC can set location (city, province)
- [ ] RCIC can set years of experience
- [ ] RCIC can set consultation rates (hourly or per-session)
- [ ] RCIC can add credentials/certifications
- [ ] Profile changes reflect immediately on public profile

**Expected Result:** RCIC profile displays comprehensive professional information to potential clients.

---

### US-2.3: RCIC Availability Management
**As an** RCIC
**I want to** set my availability schedule
**So that** clients can only book during times I'm available

**Acceptance Criteria:**
- [ ] RCIC can set weekly recurring availability (e.g., Mon-Fri 9am-5pm)
- [ ] RCIC can block specific dates (vacations, holidays)
- [ ] RCIC can set consultation duration (30min, 60min, 90min)
- [ ] RCIC can set buffer time between appointments
- [ ] Availability syncs with booking calendar
- [ ] RCIC can set timezone

**Expected Result:** Booking calendar only shows time slots when RCIC is available.

---

## 3. RCIC Directory & Search

### US-3.1: Browse RCIC Directory
**As a** Visitor or Client
**I want to** browse all approved RCICs
**So that** I can find a consultant for my immigration needs

**Acceptance Criteria:**
- [ ] Directory page lists all approved RCICs
- [ ] Each listing shows: photo, name, specialties, rating, price range, location
- [ ] Pagination or infinite scroll for large lists
- [ ] Only approved RCICs are displayed
- [ ] Default sort by rating (highest first)

**Expected Result:** User sees a list of available RCICs with key information.

---

### US-3.2: Filter RCICs
**As a** Visitor or Client
**I want to** filter RCICs by various criteria
**So that** I can narrow down consultants that match my needs

**Acceptance Criteria:**
- [ ] Filter by specialty (checkbox, multi-select)
- [ ] Filter by language (checkbox, multi-select)
- [ ] Filter by location (province/city dropdown)
- [ ] Filter by price range (slider or min/max inputs)
- [ ] Filter by minimum rating (1-5 stars)
- [ ] Filters can be combined
- [ ] Results update in real-time or on "Apply" button
- [ ] "Clear filters" option available
- [ ] URL updates with filter parameters (shareable/bookmarkable)

**Expected Result:** Directory shows only RCICs matching selected filter criteria.

---

### US-3.3: Sort RCICs
**As a** Visitor or Client
**I want to** sort the RCIC list
**So that** I can find the best consultant based on my priorities

**Acceptance Criteria:**
- [ ] Sort by rating (highest/lowest)
- [ ] Sort by price (lowest/highest)
- [ ] Sort by experience (most/least years)
- [ ] Sort by availability (soonest available)
- [ ] Default sort is by rating (highest first)
- [ ] Sort selection persists during session

**Expected Result:** RCIC list is reordered based on selected sort criteria.

---

### US-3.4: View RCIC Profile Page
**As a** Visitor or Client
**I want to** view an RCIC's detailed profile page
**So that** I can learn more about them before booking

**Acceptance Criteria:**
- [ ] Profile page displays:
  - Profile photo
  - Full name
  - Bio/About section
  - Specialties with descriptions
  - Languages spoken
  - Location
  - Years of experience
  - Consultation rates
  - Average rating (stars) and review count
  - All reviews with ratings
  - Availability calendar preview
- [ ] "Book Consultation" CTA button (redirects to login if not authenticated)
- [ ] Reviews are paginated or "load more"

**Expected Result:** Client has all information needed to decide whether to book with this RCIC.

---

## 4. Booking System

### US-4.1: View RCIC Availability Calendar
**As a** Client
**I want to** view an RCIC's availability calendar
**So that** I can see when they're available for consultation

**Acceptance Criteria:**
- [ ] Calendar displays available time slots
- [ ] Unavailable/booked slots are greyed out or hidden
- [ ] Calendar shows current week by default
- [ ] Can navigate to future weeks
- [ ] Cannot book slots in the past
- [ ] Time slots display in client's local timezone
- [ ] Minimum booking notice enforced (e.g., 24 hours ahead)

**Expected Result:** Client sees available booking slots for the selected RCIC.

---

### US-4.2: Book a Consultation
**As a** Client
**I want to** book a consultation with an RCIC
**So that** I can get professional immigration advice

**Acceptance Criteria:**
- [ ] Client selects available time slot
- [ ] Client selects consultation type:
  - Video call
  - Phone call
  - In-person (if RCIC offers)
- [ ] Client can add notes/questions for the RCIC
- [ ] Booking summary shown before payment:
  - RCIC name
  - Date and time
  - Duration
  - Consultation type
  - Price
- [ ] Client proceeds to payment
- [ ] Slot becomes unavailable for other clients immediately

**Expected Result:** Booking is created in "pending payment" status.

---

### US-4.3: Complete Booking Payment
**As a** Client
**I want to** pay for my consultation securely
**So that** my booking is confirmed

**Acceptance Criteria:**
- [ ] Stripe checkout form displayed
- [ ] Accepts major credit/debit cards
- [ ] Payment amount shows clearly
- [ ] Successful payment:
  - Booking status changes to "confirmed"
  - Confirmation page displayed with booking details
  - Confirmation email sent to client
  - Notification sent to RCIC
- [ ] Failed payment:
  - Error message displayed
  - Client can retry payment
  - Booking remains in "pending" status
- [ ] Payment is held until consultation completed (optional escrow)

**Expected Result:** Payment processed, booking confirmed, both parties notified.

---

### US-4.4: View My Bookings (Client)
**As a** Client
**I want to** view all my bookings
**So that** I can track my upcoming and past consultations

**Acceptance Criteria:**
- [ ] Dashboard shows upcoming bookings with:
  - RCIC name and photo
  - Date and time
  - Consultation type
  - Status (confirmed, completed, cancelled)
  - Join link (for video calls)
- [ ] Can view past/completed bookings
- [ ] Can filter by status
- [ ] Can cancel upcoming booking (with policy notice)

**Expected Result:** Client has complete visibility of their booking history.

---

### US-4.5: View My Bookings (RCIC)
**As an** RCIC
**I want to** view all my bookings
**So that** I can manage my consultation schedule

**Acceptance Criteria:**
- [ ] Dashboard shows today's bookings prominently
- [ ] Shows upcoming bookings with:
  - Client name
  - Date and time
  - Consultation type
  - Client's notes/questions
- [ ] Can view past/completed bookings
- [ ] Can view earnings per booking
- [ ] Can mark consultation as "completed"

**Expected Result:** RCIC can efficiently manage their consultation schedule.

---

### US-4.6: Cancel Booking
**As a** Client
**I want to** cancel an upcoming booking
**So that** I can free up the slot if my plans change

**Acceptance Criteria:**
- [ ] Cancel button available for upcoming bookings
- [ ] Cancellation policy displayed:
  - More than 24 hours: full refund
  - Less than 24 hours: 50% refund (or as configured)
- [ ] Client confirms cancellation
- [ ] Booking status changes to "cancelled"
- [ ] Refund processed automatically
- [ ] Time slot becomes available again
- [ ] Both parties notified via email

**Expected Result:** Booking cancelled, refund processed, slot reopened.

---

## 5. Payment Processing

### US-5.1: Process Consultation Payment
**As the** Platform
**I want to** process payments with automatic commission split
**So that** revenue is distributed correctly

**Acceptance Criteria:**
- [ ] Payment processed via Stripe
- [ ] 20% platform commission calculated
- [ ] 80% RCIC share calculated
- [ ] Transaction recorded with breakdown
- [ ] RCIC payout processed (immediate or scheduled)
- [ ] Platform commission tracked for analytics

**Expected Result:** Each payment is split 80/20 between RCIC and platform.

---

### US-5.2: View Earnings (RCIC)
**As an** RCIC
**I want to** view my earnings and payment history
**So that** I can track my income from the platform

**Acceptance Criteria:**
- [ ] Dashboard shows:
  - Total earnings (all time)
  - Earnings this month
  - Pending payouts
  - Completed payouts
- [ ] Transaction history with:
  - Date
  - Client name
  - Consultation type
  - Gross amount
  - Platform fee
  - Net amount received
- [ ] Can filter by date range
- [ ] Can export earnings report (CSV)

**Expected Result:** RCIC has complete visibility of their earnings and payout history.

---

### US-5.3: View Payment History (Client)
**As a** Client
**I want to** view my payment history
**So that** I can track my consultation expenses

**Acceptance Criteria:**
- [ ] Payment history shows:
  - Date
  - RCIC name
  - Amount paid
  - Payment status
  - Receipt/invoice link
- [ ] Can download receipt for each payment

**Expected Result:** Client can access records of all payments made.

---

## 6. Review & Rating System

### US-6.1: Submit Review After Consultation
**As a** Client
**I want to** rate and review my RCIC after a consultation
**So that** I can share my experience with other clients

**Acceptance Criteria:**
- [ ] Review prompt appears after consultation is marked complete
- [ ] "Rate your experience" email sent 24 hours after consultation
- [ ] Review form includes:
  - Star rating (1-5, required)
  - Text review (optional, max 1000 characters)
- [ ] Can only review after completed consultation
- [ ] Can only submit one review per consultation
- [ ] Review is published immediately (or after moderation)
- [ ] RCIC's average rating is recalculated

**Expected Result:** Review is submitted and displayed on RCIC's profile.

---

### US-6.2: View Reviews on RCIC Profile
**As a** Visitor or Client
**I want to** read reviews on an RCIC's profile
**So that** I can make an informed booking decision

**Acceptance Criteria:**
- [ ] Reviews display:
  - Client first name (privacy)
  - Star rating
  - Review text
  - Date posted
- [ ] Reviews sorted by newest first (option for highest/lowest rating)
- [ ] Average rating prominently displayed
- [ ] Total number of reviews shown
- [ ] Pagination for many reviews

**Expected Result:** User can read authentic reviews from past clients.

---

### US-6.3: View My Reviews (RCIC)
**As an** RCIC
**I want to** view all reviews I've received
**So that** I can understand client feedback

**Acceptance Criteria:**
- [ ] Dashboard shows recent reviews
- [ ] Can view all reviews with filtering options
- [ ] Shows average rating trend over time
- [ ] Cannot delete or edit reviews

**Expected Result:** RCIC has visibility into their review history and ratings.

---

## 7. Email Notifications

### US-7.1: Booking Confirmation Emails
**As a** Client and RCIC
**I want to** receive email confirmation when a booking is made
**So that** I have a record of the appointment details

**Acceptance Criteria:**
- [ ] Client receives email with:
  - Booking confirmation number
  - RCIC name and photo
  - Date, time, and timezone
  - Consultation type
  - Amount paid
  - Link to join (if video) or address (if in-person)
  - Calendar invite attachment (.ics)
- [ ] RCIC receives email with:
  - New booking notification
  - Client name
  - Date, time, and timezone
  - Consultation type
  - Client's notes/questions
  - Calendar invite attachment (.ics)

**Expected Result:** Both parties have email confirmation with all relevant details.

---

### US-7.2: Reminder Emails
**As a** Client and RCIC
**I want to** receive reminder emails before my consultation
**So that** I don't forget about the appointment

**Acceptance Criteria:**
- [ ] Reminder sent 24 hours before consultation
- [ ] Email includes:
  - Appointment details
  - Join link or location
  - Reschedule/cancel link (for client)
- [ ] Optional: Additional reminder 1 hour before

**Expected Result:** Both parties are reminded of upcoming consultation.

---

### US-7.3: Review Request Email
**As the** Platform
**I want to** send review request emails after consultations
**So that** clients are prompted to leave feedback

**Acceptance Criteria:**
- [ ] Email sent to client after consultation marked complete
- [ ] Sent within 24 hours of consultation end time
- [ ] Email includes:
  - RCIC name
  - Direct link to leave review
  - Simple call-to-action
- [ ] Only sent once per consultation
- [ ] Not sent if client already submitted review

**Expected Result:** Clients are reminded to leave reviews, improving platform content.

---

## 8. Admin Dashboard

### US-8.1: View All Users
**As an** Admin
**I want to** view all registered users
**So that** I can manage the platform's user base

**Acceptance Criteria:**
- [ ] List of all users (clients and RCICs)
- [ ] Filterable by role (client/RCIC)
- [ ] Searchable by name or email
- [ ] Shows: name, email, role, status, registration date
- [ ] Can view user details
- [ ] Can deactivate/suspend user accounts

**Expected Result:** Admin has complete visibility and control over user accounts.

---

### US-8.2: Approve/Reject RCIC Applications
**As an** Admin
**I want to** review and approve/reject RCIC applications
**So that** only qualified consultants can offer services

**Acceptance Criteria:**
- [ ] List of pending RCIC applications
- [ ] Application shows:
  - RCIC information
  - License number
  - Uploaded credentials
  - Application date
- [ ] Admin can verify credentials externally
- [ ] "Approve" action:
  - RCIC status changes to "approved"
  - RCIC receives approval email
  - RCIC profile becomes visible in directory
- [ ] "Reject" action:
  - Admin provides rejection reason
  - RCIC receives rejection email with reason
  - RCIC can reapply with corrections

**Expected Result:** Only verified RCICs are approved to offer services.

---

### US-8.3: View All Bookings
**As an** Admin
**I want to** view all bookings on the platform
**So that** I can monitor platform activity

**Acceptance Criteria:**
- [ ] List of all bookings
- [ ] Filterable by status (pending, confirmed, completed, cancelled)
- [ ] Filterable by date range
- [ ] Searchable by client or RCIC name
- [ ] Shows: booking ID, client, RCIC, date, status, amount
- [ ] Can view booking details

**Expected Result:** Admin has visibility into all booking activity.

---

### US-8.4: View Revenue Analytics
**As an** Admin
**I want to** view revenue analytics
**So that** I can track platform performance

**Acceptance Criteria:**
- [ ] Dashboard shows:
  - Total revenue (all time)
  - Revenue this month
  - Platform commission earned
  - Number of completed consultations
- [ ] Charts showing:
  - Revenue over time (daily/weekly/monthly)
  - Bookings over time
  - Top earning RCICs
- [ ] Can filter by date range
- [ ] Can export reports

**Expected Result:** Admin can track and analyze platform financial performance.

---

### US-8.5: Manage Platform Settings
**As an** Admin
**I want to** manage platform settings
**So that** I can configure how the platform operates

**Acceptance Criteria:**
- [ ] Configure commission rate (default 20%)
- [ ] Configure minimum booking notice (hours)
- [ ] Configure cancellation policy
- [ ] Manage specialty categories
- [ ] Manage supported languages list
- [ ] Enable/disable new registrations

**Expected Result:** Admin can customize platform behavior without code changes.

---

## 9. Additional Features (Nice-to-Have)

### US-9.1: RCIC Verification Badge
**As a** Client
**I want to** see a verification badge on verified RCICs
**So that** I know their credentials have been validated

**Acceptance Criteria:**
- [ ] Verified badge displays on RCIC cards and profiles
- [ ] Badge indicates admin has verified credentials
- [ ] Tooltip explains what verification means

---

### US-9.2: Favorite/Save RCICs
**As a** Client
**I want to** save RCICs to a favorites list
**So that** I can easily find them later

**Acceptance Criteria:**
- [ ] Heart/bookmark icon on RCIC cards
- [ ] Favorites list in client dashboard
- [ ] Can remove from favorites

---

### US-9.3: RCIC Response Time Display
**As a** Client
**I want to** see how quickly an RCIC typically responds
**So that** I know what to expect

**Acceptance Criteria:**
- [ ] Shows average response time on profile
- [ ] Calculated from booking confirmations

---

## Appendix: User Story Status

| ID | Story | Priority | Status |
|----|-------|----------|--------|
| US-1.1 | Client Email Registration | High | Pending |
| US-1.2 | Client Google OAuth | Medium | Pending |
| US-1.3 | RCIC Registration | High | Pending |
| US-1.4 | User Login | High | Pending |
| US-1.5 | Password Reset | Medium | Pending |
| US-2.1 | Client Profile Management | Medium | Pending |
| US-2.2 | RCIC Profile Management | High | Pending |
| US-2.3 | RCIC Availability Management | High | Pending |
| US-3.1 | Browse RCIC Directory | High | Pending |
| US-3.2 | Filter RCICs | High | Pending |
| US-3.3 | Sort RCICs | Medium | Pending |
| US-3.4 | View RCIC Profile | High | Pending |
| US-4.1 | View Availability Calendar | High | Pending |
| US-4.2 | Book Consultation | High | Pending |
| US-4.3 | Complete Payment | High | Pending |
| US-4.4 | View Bookings (Client) | Medium | Pending |
| US-4.5 | View Bookings (RCIC) | Medium | Pending |
| US-4.6 | Cancel Booking | Medium | Pending |
| US-5.1 | Payment Processing | High | Pending |
| US-5.2 | View Earnings (RCIC) | Medium | Pending |
| US-5.3 | View Payment History | Low | Pending |
| US-6.1 | Submit Review | Medium | Pending |
| US-6.2 | View Reviews | Medium | Pending |
| US-6.3 | RCIC View Reviews | Low | Pending |
| US-7.1 | Booking Confirmation Emails | High | Pending |
| US-7.2 | Reminder Emails | Medium | Pending |
| US-7.3 | Review Request Email | Low | Pending |
| US-8.1 | View All Users | Medium | Pending |
| US-8.2 | Approve/Reject RCICs | High | Pending |
| US-8.3 | View All Bookings | Medium | Pending |
| US-8.4 | Revenue Analytics | Medium | Pending |
| US-8.5 | Platform Settings | Low | Pending |