// === CASE: DeleteUsers ===
If(
    varPendingDeleteAction = "DeleteUsers",
    With(
        {
            req:  Office365Users.UserProfileV2(User().Email),
            code: "DelUser-" & Text(Now(), "yyyymmdd-hhnnss")
        },
        If(
            CountRows(colDeleteUsers) = 0,
            Notify("No users staged to delete.", NotificationType.Information),
            // Queue one Assignment row per user, skipping duplicates already pending
            ForAll(
                colDeleteUsers As u,
                If(
                    IsBlank(
                        LookUp(
                            Skill_Matrix_Assignments,
                            Lower(Employee_Email) = Lower(u.Employee_Email) &&
                            Operation.Value = "DeleteUser" &&
                            Status.Value = "Pending"
                        )
                    ),
                    Patch(
                        Skill_Matrix_Assignments,
                        Defaults(Skill_Matrix_Assignments),
                        {
                            Title:          code,
                            Operation:      { Value: "DeleteUser" },   // Choice
                            Status:         { Value: "Pending" },      // Choice
                            Employee_Email: u.Employee_Email,
                            Employee:       u.Employee,                 // optional for readability
                            RequestedAt:    Now(),
                            RequestedBy: {
                                '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                                Claims: "i:0#.f|membership|" & Coalesce(req.mail, User().Email),
                                DisplayName:   Coalesce(req.displayName, User().FullName),
                                Email:         Coalesce(req.mail, User().Email),
                                Department:    Coalesce(req.department, ""),
                                JobTitle:      Coalesce(req.jobTitle, ""),
                                Picture:       ""
                            }
                        }
                    )
                )
            );

            Notify(
                "Queued delete for " & CountRows(colDeleteUsers) & " user(s).",
                NotificationType.Success
            );

            // Cleanup UI
            Clear(colDeleteUsers);
            Reset(cmbDelUser);
            UpdateContext({
                varShowConfirmDelete: false,
                varPendingDeleteAction: "",
                varConfirmDeleteMessage: ""
            });

            // Optional: Refresh the list you display on this page
            // Refresh(Skill_Matrix_Assignments)
        )
    )
);

If(
    CountRows(colDeleteUsers) = 0,
    Notify("Choose at least one user to delete.", NotificationType.Information),
    UpdateContext({
        varPendingDeleteAction: "DeleteUsers",
        varConfirmDeleteMessage:
            "You’re about to queue deletion for " & CountRows(colDeleteUsers) & " user(s):" & Char(10) &
            Concat(
                FirstN(colDeleteUsers, 5),
                "- " & Coalesce(Employee, "") & " (" & Employee_Email & ")" & Char(10)
            ) &
            If(
                CountRows(colDeleteUsers) > 5,
                "... and " & Text(CountRows(colDeleteUsers) - 5) & " more." & Char(10),
                ""
            ) &
            Char(10) & "Proceed?",
        varShowConfirmDelete: true
    })
)
