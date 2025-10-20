If(
    varPendingDeleteAction = "DeleteModules",
    With(
        {
            req: Office365Users.UserProfileV2(User().Email),
            code: "DelMod-" & Text(Now(), "yyyymmdd-hhnnss")
        },
        ForAll(
            colDeleteModules As m,
            Patch(
                Skill_Matrix_Assignments,
                Defaults(Skill_Matrix_Assignments),
                {
                    Title:          code,
                    Operation:      { Value: "DeleteModule" },
                    Status:         { Value: "Pending" },
                    Module_Name:    m.Title,
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
        );
        Notify(
            "Queued " & CountRows(colDeleteModules) & " module delete request(s).",
            NotificationType.Success
        );
        Clear(colDeleteModules);
        Reset(cmbDelModule);
        UpdateContext({
            varShowConfirmDelete: false,
            varPendingDeleteAction: "",
            varConfirmDeleteMessage: ""
        })
    )
);
