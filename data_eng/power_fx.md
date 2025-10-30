// 1. Find added modules (in working but not in original)
ClearCollect(
    colAddedModules,
    Filter(
        colModulesInPreset_Edit_Working,
        !(Module_ID in colModulesInPreset_Edit.Module_ID)
    )
);

// 2. Find removed modules (in original but not in working)
ClearCollect(
    colRemovedModules,
    Filter(
        colModulesInPreset_Edit,
        !(Module_ID in colModulesInPreset_Edit_Working.Module_ID)
    )
);

// 3. Prep requester info + timestamp once
With(
    {
        req: Office365Users.UserProfileV2(User().Email),
        nowStr: Text(Now(), "yyyymmdd-hhnnss")
    },

    // ─────────────────────────────────────────
    // 4. APPLY CHANGES TO Skill_Matrix_Team_Presets
    //    4a. ADD any newly selected modules
    ForAll(
        colAddedModules As a,
        Patch(
            Skill_Matrix_Team_Presets,
            Defaults(Skill_Matrix_Team_Presets),
            {
                Preset_ID: varSelectedPresetId,
                Title: varSelectedPresetTitle,

                // Store BOTH name and ID
                Modules: a.ModuleName,        // readable
                Module_ID: a.Module_ID,       // GUID for flow

                IsActive: true,
                Created_By: User().Email,
                Created_At: Now()
            }
        );

        // Log the AddModule assignment (only if not already pending)
        If(
            IsBlank(
                LookUp(
                    Skill_Matrix_Assignments,
                    Team_Preset_ID = varSelectedPresetId &&
                    Operation.Value = "AddModule" &&
                    Module_Name = a.ModuleName &&
                    Status.Value = "Pending"
                )
            ),
            Patch(
                Skill_Matrix_Assignments,
                Defaults(Skill_Matrix_Assignments),
                {
                    Title: "AddMod-" & nowStr,

                    Team_Preset: varSelectedPresetTitle,
                    Team_Preset_ID: varSelectedPresetId,

                    // 👇 keep these columns/names the same as today
                    Operation: { Value: "AddModule" },
                    Status: { Value: "Pending" },

                    Module_Name: a.ModuleName,
                    Module_ID: a.Module_ID,

                    RequestedAt: Now(),
                    RequestedBy: {
                        '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                        Claims: "i:0#.f|membership|" & Lower(Coalesce(req.mail, User().Email)),
                        DisplayName: Coalesce(req.displayName, User().FullName),
                        Email: Coalesce(req.mail, User().Email),
                        Department: Coalesce(req.department, ""),
                        JobTitle: Coalesce(req.jobTitle, ""),
                        Picture: ""
                    }
                }
            )
        )
    );

    //    4b. REMOVE any modules that were deselected
    ForAll(
        colRemovedModules As r,
        // 4b-i) Mark preset row inactive instead of hard-delete
        Patch(
            Skill_Matrix_Team_Presets,
            LookUp(
                Skill_Matrix_Team_Presets,
                Preset_ID = varSelectedPresetId &&
                Module_ID = r.Module_ID
            ),
            {
                IsActive: false
            }
        );

        // 4b-ii) Log the RemMod assignment (always mark Completed here, same as your original)
        Patch(
            Skill_Matrix_Assignments,
            Defaults(Skill_Matrix_Assignments),
            {
                Title: "RemMod-" & nowStr,

                Team_Preset: varSelectedPresetTitle,
                Team_Preset_ID: varSelectedPresetId,

                // 👇 keep existing semantics
                Operation: { Value: "RemoveModule" },
                Status: { Value: "Completed" },

                Module_Name: r.ModuleName,
                Module_ID: r.Module_ID,

                RequestedAt: Now(),
                RequestedBy: {
                    '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                    Claims: "i:0#.f|membership|" & Lower(Coalesce(req.mail, User().Email)),
                    DisplayName: Coalesce(req.displayName, User().FullName),
                    Email: Coalesce(req.mail, User().Email),
                    Department: Coalesce(req.department, ""),
                    JobTitle: Coalesce(req.jobTitle, ""),
                    Picture: ""
                }
            }
        )
    );
);

// 5. Refresh baseline snapshot so diffs reset
ClearCollect(
    colModulesInPreset_Edit,
    colModulesInPreset_Edit_Working
);

// 6. Notify user
Notify(
    "Preset changes saved. " &
    "Added: " & CountRows(colAddedModules) &
    " | Removed: " & CountRows(colRemovedModules),
    NotificationType.Success
);

// 7. Reset the dropdown and clear pending selection
Reset(cmbAddModules_Edit);
UpdateContext({varModuleToRemove: ""});
UpdateContext({varConfirmRemove: false});
