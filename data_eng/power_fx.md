// 4. Soft-remove (IsActive=false) for anything in colRemovedModules
ForAll(
    colRemovedModules As gone,
    ForAll(
        Filter(
            Skill_Matrix_Team_Presets,
            Preset_ID = varSelectedPresetId,
            IsActive = true,
            // match either by Module_ID or by name fallback
            (Module_ID = gone.Module_ID) ||
            (Modules = gone.ModuleName)
        ) As rowToClose,
        Patch(
            Skill_Matrix_Team_Presets,
            rowToClose,
            {
                IsActive: false,
                Updated_By: User().Email,
                Updated_At: Now()
            }
        )
    )
);

// 5. Insert new active rows in Team_Presets for anything in colAddedModules
ForAll(
    colAddedModules As newMod,
    Patch(
        Skill_Matrix_Team_Presets,
        Defaults(Skill_Matrix_Team_Presets),
        {
            Preset_ID: varSelectedPresetId,
            Title: varSelectedPresetTitle,

            // human-readable module name
            Modules: newMod.ModuleName,

            // machine id
            Module_ID: newMod.Module_ID,

            IsActive: true,
            Created_By: User().Email,
            Created_At: Now()
        }
    )
);

// 6. Queue Assignments rows for Flow

// 6a. Added modules → AddModule operation
If(
    CountRows(colAddedModules) > 0,
    With(
        {
            codeAdd: "AddMod-" & Text(Now(), "yyyymmdd-hhnnss"),
            req: Office365Users.UserProfileV2(User().Email)
        },
        Patch(
            Skill_Matrix_Assignments,
            Defaults(Skill_Matrix_Assignments),
            {
                Title: codeAdd,
                Operation: { Value: "AddModule" },
                Status: { Value: "Pending" },

                // which preset is being edited
                Team_Preset_ID: varSelectedPresetId,
                Team_Preset_Name: varSelectedPresetTitle,

                // all module IDs that were added (semicolon list)
                Module_ID: Concat(colAddedModules, Module_ID, ";"),
                Module_Name: Concat(colAddedModules, ModuleName, ";"),

                RequestedAt: Now(),
                RequestedBy: {
                    '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                    Claims: "i:0#.f|membership|" & Coalesce(req.mail, User().Email),
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

// 6b. Removed modules → RemModule operation
If(
    CountRows(colRemovedModules) > 0,
    With(
        {
            codeRem: "RemMod-" & Text(Now(), "yyyymmdd-hhnnss"),
            req2: Office365Users.UserProfileV2(User().Email)
        },
        Patch(
            Skill_Matrix_Assignments,
            Defaults(Skill_Matrix_Assignments),
            {
                Title: codeRem,
                Operation: { Value: "RemModule" },
                Status: { Value: "Pending" },

                Team_Preset_ID: varSelectedPresetId,
                Team_Preset_Name: varSelectedPresetTitle,

                Module_ID: Concat(colRemovedModules, Module_ID, ";"),
                Module_Name: Concat(colRemovedModules, ModuleName, ";"),

                RequestedAt: Now(),
                RequestedBy: {
                    '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                    Claims: "i:0#.f|membership|" & Coalesce(req2.mail, User().Email),
                    DisplayName: Coalesce(req2.displayName, User().FullName),
                    Email: Coalesce(req2.mail, User().Email),
                    Department: Coalesce(req2.department, ""),
                    JobTitle: Coalesce(req2.jobTitle, ""),
                    Picture: ""
                }
            }
        )
    )
);

// 7. Refresh + resnap so UI matches SharePoint after save
Refresh(Skill_Matrix_Team_Presets);

// rebuild Original from SharePoint so next edit starts clean
ClearCollect(
    colModulesInPreset_Original,
    ForAll(
        Filter(
            Skill_Matrix_Team_Presets,
            Preset_ID = varSelectedPresetId,
            IsActive = true
        ) As row2,
        With(
            {
                rId:
                    If(
                        !IsBlank(row2.Module_ID),
                        row2.Module_ID,
                        LookUp(
                            Skill_Matrix_Modules,
                            Title = row2.Modules,
                            Mod_ID
                        )
                    ),
                rName:
                    If(
                        !IsBlank(row2.Modules),
                        row2.Modules,
                        LookUp(
                            Skill_Matrix_Modules,
                            Mod_ID = row2.Module_ID,
                            Title
                        )
                    )
            },
            {
                Module_ID: rId,
                ModuleName: rName
            }
        )
    )
);

// working copy = new truth
ClearCollect(
    colModulesInPreset_Edit_Working,
    colModulesInPreset_Original
);

// clear diff collections
Clear(colAddedModules);
Clear(colRemovedModules);

// final toast to supervisor
Notify(
    "Saved. Added " & CountRows(colAddedModules) &
    " | Removed " & CountRows(colRemovedModules),
    NotificationType.Success
);






With(
    {
        currentIDs: AddColumns(
            colModulesInPreset_Edit_Working,
            keepId,
            Module_ID
        )
    },
    SortByColumns(
        Filter(
            colAllModules,
            // keep only modules not already in the working preset
            IsBlank(
                LookUp(
                    currentIDs,
                    keepId = Mod_ID
                )
            )
        ),
        "Title",
        SortOrder.Ascending
    )
)



If(
    !IsBlank(cmbAddModules_Edit.Selected),
    With(
        {
            mId: cmbAddModules_Edit.Selected.Mod_ID,
            mName: cmbAddModules_Edit.Selected.Title
        },
        If(
            IsBlank(
                LookUp(
                    colModulesInPreset_Edit_Working,
                    Module_ID = mId
                )
            ),
            Collect(
                colModulesInPreset_Edit_Working,
                {
                    Module_ID: mId,
                    ModuleName: mName
                }
            )
        )
    );
    Reset(cmbAddModules_Edit)
);
