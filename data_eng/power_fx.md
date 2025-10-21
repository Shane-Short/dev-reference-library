// Handle confirm actions for the modal
Switch(
    varPendingAction,

    // ========== CASE: Assign + Unassign together ==========
    "AssignUnassign",
    With(
        {
            req:  Office365Users.UserProfileV2(User().Email),
            code: "Assign-" & Text(Now(), "yyyymmdd-hhnnss")
        },

        If(
            CountRows(colAddedUsers) = 0 && CountRows(colRemovedUsers) = 0,
            Notify("No changes to queue.", NotificationType.Information),

            // ---- Queue ADDs (SeedUser vs ReseedUser) ----
            ForAll(
                colAddedUsers As a,
                With(
                    {
                        // treat as known if they already have any User_Settings row for this preset
                        knownForThisPreset:
                            LookUp(
                                Skill_Matrix_User_Settings,
                                Lower(Employee_Email) = Lower(a.Employee_Email) &&
                                Team_Preset_ID = varPresetId
                            )
                    },
                    Patch(
                        Skill_Matrix_Assignments,
                        Defaults(Skill_Matrix_Assignments),
                        {
                            Title:          code,
                            Employee_Email: a.Employee_Email,
                            Team_Preset:    varPresetName,
                            Team_Preset_ID: varPresetId,
                            Operation:      { Value: If(IsBlank(knownForThisPreset), "SeedUser", "ReseedUser") },
                            Status:         { Value: "Pending" },
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

            // ---- Queue REMOVEs (RemoveUserFromPreset) ----
            ForAll(
                colRemovedUsers As r,
                Patch(
                    Skill_Matrix_Assignments,
                    Defaults(Skill_Matrix_Assignments),
                    {
                        Title:          code,
                        Employee_Email: r.Employee_Email,
                        Team_Preset:    varPresetName,
                        Team_Preset_ID: varPresetId,
                        Operation:      { Value: "RemoveUserFromPreset" },
                        Status:         { Value: "Pending" },
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

            // ---- Optimistic UI: reflect adds now in User_Settings (keeps page in sync) ----
            ForAll(
                colAddedUsers As a,
                Patch(
                    Skill_Matrix_User_Settings,
                    Coalesce(
                        LookUp(
                            Skill_Matrix_User_Settings,
                            Lower(Employee_Email) = Lower(a.Employee_Email) &&
                            Team_Preset_ID = varPresetId
                        ),
                        Defaults(Skill_Matrix_User_Settings)
                    ),
                    {
                        Title:          varPresetName,     // if Title stores Preset name
                        Employee:       a.Employee,
                        Employee_Email: a.Employee_Email,
                        Team_Preset:    varPresetName,
                        Team_Preset_ID: varPresetId,
                        Timestamp:      Now()
                        // Or Text(Now(), "[$-en-US]yyyy-mm-dd hh:mm:ss") if your column is Text
                    }
                )
            );

            // Bring "original" = "working" so next diff compares to new baseline
            Clear(colPresetUsersOriginal);
            Collect(colPresetUsersOriginal, colPresetUsersWorking);

            // Toast + close modal
            Notify(
                "Queued adds: " & CountRows(colAddedUsers) &
                " | removals: " & CountRows(colRemovedUsers) & ".",
                NotificationType.Success
            )
        );

        // Common cleanup for this case
        UpdateContext({ varShowConfirm: false, varPendingAction: "", varConfirmMessage: "" });
        Clear(colAddedUsers);
        Clear(colRemovedUsers);

        // Optional: refresh lists visible on screen
        Refresh(Skill_Matrix_Assignments);
        Refresh(Skill_Matrix_User_Settings);

        // Optional: reset pickers
        // Reset(cmbPeople);

    ),

    // ========== DEFAULT (safety) ==========
    Notify("No action selected.", NotificationType.Warning)
);
