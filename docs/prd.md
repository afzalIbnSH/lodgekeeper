I'm building this for my dad's lodge.

At a high level, the bookings are all offline and will continue to do so. The purpose of the software is to streamline the management of the offline bookings and related accounting. We also need to support reserving future bookings which are usually made via phone calls.

High level internal notes:
- Keep things minimal
- Online bookings are out of scope.
- There are no plans to actively pursue other clients, so multi-tenancy is out of scope. Expected scale is at most 700 bookings and 1000 guests per month. If at all a concrete second client come to happen in the future, adding multi-tenancy and the required data isolation before giving them access shouldn't be hard.
- Although the lodge currently has only full-room rentals, use the term "unit" instead of "room" so that it is easier to scale.

Below are what the client expect from the software;

- Securely login. Internal note: We will use a straightforward username/password login at this point, provisioned by a script or admin API. There won't be a self-serve option.

- Configure units with their names and their rates. Internal notes:
    - Seed unit types and associated units from the backend using a script or an admin API. Only related self-serve option will be to edit their rates.

- See available units and create a booking for approaching guest or group of guests (multiple guests support is important).
    - Internal note: Derive availability from bookings, don't use an `is_occupied` flag and introduce a second source of truth.
    - Don't allow conflicting bookings for the same unit, even if two bookings are being saved at the same time.
    - At actual check-in, there must be at least one guest. Fill in the name, ID-type (Aadhar / Passport / License or Election ID) and ID-number for every guest, along with the address, purpose of visit, check in time and number of days for the booking.
        - Capture exactly one guest's address even when there are multiple guests. There shouldn't be an option to capture more addresses. Internal note: To simplify, save address at the core booking table (assuming we use an associated table to save the guests).
        - Internal note: A reusable guest profile is out of scope.
    - Capture and upload the ID picture for every guest at actual check-in.
        - Internal note: Use S3 to store images.
    - See rent of the unit pre-filled based on the pre-configured amount against the unit type and selected number of days
        - Allow the total rent for the whole stay to be edited.
        - Changing the default rate later shouldn't change the rent already agreed for a saved booking.
        - Keep all amounts exact to the paise (INR). The amount should be broken down into base, CGST and SGST, adding up exactly to the chosen total. Both GST amounts should be 2.5% of the base amount. To calculate this; if chosen amount for example is 600;
            - Calculate base amount as 600/1.05 = 571.428571429. Truncate this to two decimal places, which in this case would be 571.42.
            - Compute the GSTs by doing (600 - 571.42) / 2 = 14.29 each.
            - If the tax amounts has more than 2 decimal places, for example 14.293, round upward to two decimal places, to 14.30 in this example, and adjust the final base to preserve the total (so that the taxes never become less than 2.5% but only possibly fractionally higher).
        Internal note: Changing GST values is out of scope. Simply use a code-level default.
    - Input paid amounts, in full or partial. Full pay is usually taken upfront but at times, especially a long staying guest, can pay in installments. So need to capture every associated payments individually with timestamps. Re-visiting a booking must show the payments and pending amount if any.
    - Compute checkout time as 24 hours * number of selected days after check-in
        - Allow the time to be edited
        - Internal note: Non-rolling checkouts are out of scope.

- Accept future bookings.
    - Assign a particular unit when recording the reservation. Reservations for just a unit type are out of scope.
    - A future booking can be planned after the current guest's expected checkout. But the unit stays occupied until the guest actually checks out, even if they're late.
    - Still allow the actual checkout time to be saved if a guest leaves late, even if it runs into the time reserved for the next guest who hasn't checked in yet.
    - Capture the caller's name and phone number. The caller doesn't have to be one of the guests.
    - Guest details, IDs, the shared address and purpose of visit can wait until actual check-in.
    - Allow a reservation to be cancelled before actual check-in. Keep the cancelled booking, but free up the unit for that period.
    - Internal note: Let this be derived in db via a scheduled check in time and a null actual check in time. Don't use a separate `booking_status` column and introduce a second source of truth.

- When guest/s checkout;
    - Mark the booking as checked out, which should save the actual checked out time in the booking. Internal note: Again, don't use a separate `booking_status` column, just have a actual check out column which shall be null originally and filled on check out.

- See occupied units, with earliest checkout seen at top (allow sorting the opposite way as well)

- Able to generate and print out receipt of a booking with all tenant and booking details.
    - By default, avoid id details and pic. But allow to optionally add them for any possible legal purposes.

- Able to generate and print monthly ledger to use for monthly gst submission. Internal note: Keep the backend of the ledger generation open ended where any start and end dates are accepted to allow scaling to other date ranges. The format below;

| Unit | Rent | CGST | SGST |
| :--- | :---: | :---: | :---: |
| 201 | 571.42 | 14.29 | 14.29 |
| 202 | 571.42 | 14.29 | 14.29 |
| 203 | 571.42 | 14.29 | 14.29 |
| **Total:** | **1,714.26** | **42.87** | **42.87** |
