## Data storage
### Users
Users can be either **initialised** or **uninitialised**, depending on whether or not they have gone through the onboarding process. Uninitialised users have no associated data besides their authentication credentials; initialised users must have all the data collected during onboarding. All users start as uninitialised and become initialised upon completion of onboarding.

Initialised and uninitialised users are stored in separate database tables:

|initialised_users   |
|--------------------|
|name: text          |
|onboarded: date     |
|*[auth credentials]*|

<br>

|uninitialised_users |
|--------------------|
|*[auth credentials]*|

The user's name is stored here as a single string (preferably in Unicode) for [maximum flexibility](http://www.kalzumeus.com/2010/06/17/falsehoods-programmers-believe-about-names/). The authentication credentials (or part of them, such as a username) ought to serve as a suitable key; otherwise, the usual practice of adding an auto-generated `id` field can be adopted.

### Medications
To each medication will be assigned a unique identifier. A numerical ID, or one that is otherwise abstracted from the name of the medication, is best for this purpose since we can then guarantee that it will not be subject to change (or misspelling, or other issues associated with the use of free text). However, it is clearly still important for humans to know which medication is referred to by each ID, and so the back-end database should include a table which, at the very least, associates a natural-language name to each ID:

|medications|
|-----------|
|id: integer|
|name: text |

Note, however, that this is a presentational nicety; the business logic will use the abstract IDs exclusively. The disadvantage of this approach is that, in order to provide a human-readable list of medications for the user to choose from, client applications must possess a copy of the above table. Fortunately, changes to this table are likely to be small and infrequent, and mainly comprising additions of new medications rather than alterations to existing ones. It would suffice, then, for clients to poll for updates from the back end every fortnight, say, or simply whenever the user wishes to update their prescription details.

### Prescriptions

Every user's entire prescription history must be stored; any update to prescription details must augment that history and not alter it. To this end, prescription information should be annotated with a start and end date, to indicate the time interval during which it was applicable:

|prescriptions                           |
|----------------------------------------|
|user: foreign key to `initialised_users`|
|medication: foreign key to `medications`|
|started: date                           |
|finished: date                          |
|dosage: float + unit                    |

(Depending on the sophistication of the database engine, it may not be possible to store numerical data annotated with units of measurement, in which case `unit` would need to be a separate column.)

A user's current prescription, however, will not have an associated end date. A simple solution would be to permit nullability of the `finished` column, but this introduces validity concerns, since we would need somehow to ensure that `finished` was not null more than once for a given user (which would indicate more than one "current" prescription). Another solution would be to have an additional table `current_prescriptions`, lacking the `finished` column and keyed only on user ID. Whenever a user changed prescription, their corresponding row in `current_prescriptions` would be transferred to `prescriptions` with a finish date added, and the information pertaining to their new prescription would be inserted into `current_prescriptions` in its place.

In principle, the `finished` field is unnecessary: if all of a given user's prescriptions are sorted into chronological order by `started`, then the finish date of each prescription should be the same as the start date of the one following it. However, the inclusion of both dates allows us to see if any gaps have been introduced into a user's prescription history, perhaps by data loss or application error. It is important that the right semantics be assigned to such gaps: they should mean only that information is missing or unavailable for the period over which they span, not that the user was not prescribed any medication during that period. The latter, after all, is a piece of information in itself, rather than a lack of information. This requires that we have some way of recording "no medication" in the `prescriptions` table. One option is to make the `medication` column nullable; another is to introduce a special medication ID for this case.

### Check-in/survey results
Since users' answers to the check-in questions are gathered all at once (and similarly for the survey questions), we are justified in storing these answers together in a single table. This still permits some flexibility regarding the exact set of questions asked, since the table schema can be adjusted accordingly; if the questions had a more complex flow, or if some of them could be left unanswered upon submission, then the aforementioned approach would be inadequate.

For the fortnightly survey:

|fortnightly_results|
|-------------------|
|date: date        |
|memory_organisation: integer|
|memory_concentration: integer|
|memory_date: integer|
|memory_phone: integer|
|memory_blank: integer|
|health_q1: integer|
|health_q2: integer|
|health_q3: integer|
|health_q4: integer|
|health_q5: integer|
|health_q6: integer|
|health_q7: integer|
|health_q8: integer|
|health_q9: integer|

(Validation of ranges may be available via the database engine, but must otherwise be enforced in the business logic of the back end.)

The daily survey results table will look much the same, but the situation is slightly more complicated since the user is asked to provide a (possibly empty) list of side effects they have experienced. This will require a separate table:

|daily_side_effects|
|------------------|
|user: foreign key to `initialised_users`|
|date: date        |
|side_effect: text |

(Instead of storing the side effects as free text, we could take the same approach as with medications and assign each common side effect a unique ID. The `side_effect` field in `daily_side_effects` would then contain a foreign key to a table which associated each ID to a text description of the effect. The above approach, however, is not only simpler, but also allows user-defined side effects to reside in the same table.)