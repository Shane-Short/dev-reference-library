// 0. Safety reset for downstream UX state
Set(varConfirmRemove, false);
Set(varModuleToRemove, "");

// 1. Cache the chosen preset id + title for downstream Save button
Set(varSelectedPresetId, cmbEditPreset.Selected.Value);
Set(varSelectedPresetTitle, cmbEditPreset.Selected.Title);

// 2. Build ORIGINAL snapshot = all ACTIVE modules currently in that preset
//    Shape rows to { Module_ID: "...", ModuleName: "..." }
ClearCollect(
    colModulesInPreset_Original,
    ForAll(
        Filter(
            Skill_Matrix_Team_Presets,
            Preset_ID = varSelectedPresetId,
            IsActive = true
        ) As row,
        With(
            {
                resolvedId:
                    If(
                        !IsBlank(row.Module_ID),
                        row.Module_ID,
                        LookUp(
                            Skill_Matrix_Modules,
                            Title = row.Modules,
                            Mod_ID
                        )
                    ),
                resolvedName:
                    If(
                        !IsBlank(row.Modules),
                        row.Modules,
                        LookUp(
                            Skill_Matrix_Modules,
                            Mod_ID = row.Module_ID,
                            Title
                        )
                    )
            },
            {
                Module_ID: resolvedId,
                ModuleName: resolvedName
            }
        )
    )
);

// 3. Working copy starts identical to ORIGINAL snapshot
ClearCollect(
    colModulesInPreset_Edit_Working,
    colModulesInPreset_Original
);

// 4. Reset helper collections we use in the Save button’s Notify
Clear(colAddedModules);
Clear(colRemovedModules);

// 5. Reset the "add module" combobox
Reset(cmbAddModules_Edit);





// 1. Recalculate Added and Removed
//    Added = in Working but NOT in Original
ClearCollect(
    colAddedModules,
    Filter(
        colModulesInPreset_Edit_Working,
        IsBlank(
            LookUp(
                colModulesInPreset_Original,
                Module_ID = ThisRecord.Module_ID
            )
        )
    )
);

//    Removed = in Original but NOT in Working
ClearCollect(
    colRemovedModules,
    Filter(
        colModulesInPreset_Original,
        IsBlank(
            LookUp(
                colModulesInPreset_Edit_Working,
                Module_ID = ThisRecord.Module_ID
            )
        )
    )
);

// 2. Prepare requester info for Assignments rows
With(
    {
        req: Office365Users.UserProfileV2(User().Email),
        nowStamp: Text(Now(), "yyyymmdd-hhmmss")
    },

    // 3A. Handle ADDED modules for this preset
    ForAll(
        colAddedModules As a,
        // Write/activate row in Team_Presets for this preset+module
        Patch(
            Skill_Matrix_Team_Presets,
            Defaults(Skill_Matrix_Team_Presets),
            {
                Preset_ID: varSelectedPresetId,
                Title: varSelectedPresetTitle,
                Module_ID: a.Module_ID,
                Modules: a.ModuleName,
                IsActive: true,
                Created_By: User().Email,
                Created_At: Now()
            }
        );

        // Queue “AddModule” work for Flow in Assignments
        Patch(
            Skill_Matrix_Assignments,
            Defaults(Skill_Matrix_Assignments),
            {
                Title: "AddMod-" & nowStamp,
                Team_Preset: varSelectedPresetTitle,
                Team_Preset_ID: varSelectedPresetId,

                Operation: {Value: "AddModule"},

                Module_Name: a.ModuleName,
                Module_ID: a.Module_ID,

                Status: {Value: "Pending"},
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

    // 3B. Handle REMOVED modules for this preset
    ForAll(
        colRemovedModules As r,
        // Flip IsActive = false on that preset-module pair
        Patch(
            Skill_Matrix_Team_Presets,
            LookUp(
                Skill_Matrix_Team_Presets,
                Preset_ID = varSelectedPresetId &&
                Module_ID = r.Module_ID &&
                IsActive = true
            ),
            {
                IsActive: false
            }
        );

        // Queue “RemoveModule” work for Flow in Assignments
        Patch(
            Skill_Matrix_Assignments,
            Defaults(Skill_Matrix_Assignments),
            {
                Title: "RemMod-" & nowStamp,
                Team_Preset: varSelectedPresetTitle,
                Team_Preset_ID: varSelectedPresetId,

                Operation: {Value: "RemoveModule"},

                Module_Name: r.ModuleName,
                Module_ID: r.Module_ID,

                Status: {Value: "Completed"}, // matches what you had before
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

// 4. Refresh baseline so everything matches post-save
ClearCollect(
    colModulesInPreset_Original,
    colModulesInPreset_Edit_Working
);

// 5. Toast summary using those diff collections we just built
Notify(
    "Preset changes saved.  Added: " &
        CountRows(colAddedModules) &
        "  /  Removed: " &
        CountRows(colRemovedModules),
    NotificationType.Success
);

// Optional: reset the preset dropdown if you like
// Reset(cmbEditPreset);
