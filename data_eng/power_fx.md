If(
    CountRows(colDeletePresets) = 0,
    Notify("Select at least one preset to delete.", NotificationType.Information),
    UpdateContext({
        varPendingDeleteAction: "DeletePresets",
        varConfirmDeleteMessage:
            "You’re about to queue deletion for " &
            CountRows(colDeletePresets) & " preset(s). This will remove user assignments and preset rows. Proceed?",
        varShowConfirmDelete: true
    })
)


// === CASE: DeletePresets ===
If(
    varPendingDeleteAction = "DeletePresets",
    With(
        {
            req:  Office365Users.UserProfileV2(User().Email),
            code: "DelPre-" & Text(Now(), "yyyymmdd-hhnnss")
        },
        ForAll(
            colDeletePresets As p,
            Patch(
                Skill_Matrix_Assignments,
                Defaults(Skill_Matrix_Assignments),
                {
                    Title:          code,
                    Operation:      { Value: "DeletePreset" }, // Choice
                    Status:         { Value: "Pending" },      // Choice
                    Team_Preset:    p.Preset_Name,
                    Team_Preset_ID: p.Preset_ID,
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
            "Queued " & CountRows(colDeletePresets) & " preset delete request(s).",
            NotificationType.Success
        );
        Clear(colDeletePresets);
        Reset(cmbDelPreset);
        UpdateContext({ varShowConfirmDelete:false, varPendingDeleteAction:"", varConfirmDeleteMessage:"" })
    )
);
