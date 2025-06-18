# How People in the App are Connected to Their Companies: A Simple Guide

Imagine our application is like a big, modern office building. This building is home to many different companies, each with its own office space. The people using our app are like the employees and bosses working in these offices.

This guide explains how we know which person belongs to which company office inside our big building.

## Step 1: Everyone Gets an ID Badge (Your Basic User Account)

When someone first starts using our app, whether they're a boss starting a new company or an employee being invited, they get a basic **ID Badge**.

*   Think of this like a standard security badge for the building.
*   It mainly says *who the person is* (like their name and email).
*   At this very first step, the badge itself doesn't say which company office they work for yet.

*(In tech terms: This is their `auth.users` record in Supabase.)*

## Step 2: The All-Important Profile Form (The `profiles` Table)

Along with their ID Badge, everyone has a **Profile Form**. This form is linked to their ID badge and keeps more detailed information about them.

One of the most important boxes on this Profile Form is:

*   **"Which Company Office Do You Belong To?"**

This box is where we record the unique name or number of the company office that person is part of.

*(In tech terms: This is their record in the `profiles` table, and the "Which Company Office Do You Belong To?" box is the `company_id` field.)*

## Step 3: The Building's Company Directory (The `companies` Table)

Our office building has a main **Company Directory**. This directory lists all the company offices that have space in the building.

For each company office, the directory knows:

*   Its unique name or number (this is its `company_id`).
*   Who the main boss or owner of that office is (this is the `owner_id`, which is the ID Badge number of the boss).

*(In tech terms: This is the `companies` table.)*

## How Do People Get Their "Company Office" Filled In on Their Profile Form?

This is the key part! How does the app know which company office to write down in that important box on each person's Profile Form? It happens in a couple of ways:

**A. The First Person from a Company (The Boss Starting a New Office)**

1.  **Setting Up Shop**: Imagine Sarah wants to start her new business, "Sarah's Super Services," in our office building. She's the first one from her company.
2.  **Gets Her Badge**: Sarah signs up for the app and gets her basic ID Badge. Initially, the "Company Office" box on her Profile Form is empty.
3.  **Registers Her Company Office**: Sarah then goes through a "Company Setup" process in the app.
    *   She tells the building management (the app) about her new company: "I'm opening Sarah's Super Services."
    *   This creates a new listing in the main Company Directory for "Sarah's Super Services" and notes Sarah as the boss (`owner_id`). Her company gets a unique office name/number.
4.  **Profile Form Updated**: The app then automatically fills in the "Company Office" box on Sarah's *own* Profile Form with the unique name/number of "Sarah's Super Services."

Now, the app knows Sarah belongs to "Sarah's Super Services" and is its boss.

*(In tech terms: This involves processes likely in `js/agency_setup.js` and the `save-company-details` function, which create a record in the `companies` table and then update the admin's `profiles.company_id`.)*

**B. New Employees Invited to an Existing Company Office (Invited Staff)**

1.  **The Invitation Letter**: Sarah, the boss of "Sarah's Super Services," wants to hire Tom.
    *   Sarah uses the app to "Send an Invitation" to Tom.
    *   Think of this as an official invitation letter. Importantly, this letter has some hidden notes on it, and one of those notes is the unique office name/number for "Sarah's Super Services."
2.  **Tom Arrives**: Tom gets the invitation email and clicks the special link inside. This link takes him to the building's reception (the app's `set-password.html` page) to get his own ID Badge and choose a password.
3.  **Automatic Profile Update**: As Tom is getting his ID Badge and setting his password, the system is smart:
    *   It reads the hidden notes on his invitation letter (the metadata).
    *   It automatically finds the "Company Office" box on Tom's new Profile Form.
    *   It fills in this box with "Sarah's Super Services" unique office name/number, taken directly from those hidden notes on the invitation.

So, from the moment Tom finishes setting his password, the app knows he works for "Sarah's Super Services." He didn't have to figure it out or choose it; the invitation system handled it.

*(In tech terms: The `invite-staff-member` Edge Function embeds the admin's `company_id` into the invitation metadata. When Tom signs up, the `handle_new_user_profile_setup` database trigger reads this metadata from `auth.users.raw_user_meta_data` and updates Tom's `profiles.company_id`.)*

## Why Is Knowing "Which Office You Belong To" So Important?

This system of linking people to company offices is super important for a few reasons:

1.  **Keeping Things Tidy**: Everyone knows where they belong. Work for "Sarah's Super Services" stays with "Sarah's Super Services."
2.  **Privacy and Security**:
    *   This is the most important reason! It means Tom, who works for "Sarah's Super Services," can only see and work on things (like customer lists, tasks, properties) that belong to "Sarah's Super Services."
    *   He can't accidentally see or change information for "David's Design Den" in the office next door.
    *   It's like Tom's ID Badge will only unlock the doors and filing cabinets within his own company's office suite. The app checks his Profile Form to see which office he belongs to before letting him see or do anything.
3.  **Doing the Right Work**: When Tom creates something new in the app (like a task for a client of "Sarah's Super Services"), the app automatically knows this new task belongs to "Sarah's Super Services" because it looks at Tom's Profile Form.

So, that little "Which Company Office Do You Belong To?" box on everyone's Profile Form is a small but mighty tool that keeps everything in our big office building app organized and secure for everyone!
