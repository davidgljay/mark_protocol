1xx - Positive Chitt Update
 - The user has earned additional trust and is linking to a new chitt as a result. Viewers of this chitt should be aware of the increased trust that the user has earned (e.g. an employee has been promoted.)
2xx - Positive Context
 - A note is being added to the chitt that indicates that the user is deserving of additional trust, but no change to the chitt is taking place.
3XX - Neutral Update
 - There is a neutral update to the chitt, such as a refresh of a valid_until date.
 - This update is relevant to someone understanding the trustworthiness of the chitt but is neither positive nor negative (e.g. this person remains an active member of our community.) 
4xx - Neutral Context
 - Information is being added which is neutral but pertinent context for verifiers.
5xx - Programmatic Update
 - Updates to the chitt that are triggered automatically for programmatic reasons and bear no reflection on the trustworthiness of the chitt.
6XX - Negative Context
 - A note that could imply that the chitt is less trustworthy, but that does not warrant revocation.
7XX - Negative Update
 - The chitt is being updated in ways that reduce its privileges (e.g. admin rights are being removed).
 - This may or may not imply that the chitt holder is less trustworthy, this can be expressed by subcode (e.g. a 710 may mean that some has termed out of a particular responsibility but is still in good standing, while a 760 may be more of a dishonorable discharge.)
 - In general, lower numbers imply more trustworthy, higher numbers imply less trustworthy, and numbers in the middle are neutral. So a 750 would be “rights are being removed from this chitt for entirely procedural reasons” while a 710 would be “this person is retiring their rights after exemplary service”.
8XX - Quiet revocation
 - This chitt is being revoked. Anyone directly accessing this chitt should know about it, but the chitt holder does not pose an active risk to other communities.
 - E.g. the chitt holder is leaving a job. The key of only this chitt has been compromised.
9xx - Loud Revocation
 - This chitt is being revoked, and the holder of the chitt may pose risks to other communities who should be noted accordingly.
 - If someone runs a community where people show multiple chitts and a member gets a 9XX revocation, the individual involved may want to consider passing the revocation on to issuers of other chitts that they have seen this person use.
 - E.g. This person’s entire wallet has been compromised. This person is a bad actor utilizing this chitt under false and potentially harmful pretenses.