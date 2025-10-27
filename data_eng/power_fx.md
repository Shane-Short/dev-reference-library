// ----------------------
// 0) Context flags
// ----------------------
Set(varHasModuleContext, !IsBlank(varSelectedModuleId) || Len(Trim(txtNewModuleName.Text)) > 0);

// ----------------------
// 1) Build CatItem universe + BEFORE snapshot for Skill_Type
// ----------------------
ClearCollect(
    colCIUniverse,
    ShowColumns(
        If(
            !IsBlank(drpCategory.Selected.Value),
            Filter(Skill_Matrix_CategoryItems, Category = drpCategory.Selected.Value),
            Skill_Matrix_CategoryItems
        ),
        "CatItem_ID"
    )
);
ClearCollect(
    colCIIds,
    ShowColumns(GroupBy(colCIUniverse, "CatItem_ID", "grp"), "CatItem_ID")
);
ClearCollect(
    colCIBefore,
    ForAll(
        colCIIds As t,
        {
            CatItem_ID: t.CatItem_ID,
            Skill_Type: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = t.CatItem_ID, Skill_Type)
        }
    )
);

// ----------------------
// 2) If module context: resolve/create module and sync Reference links
// ----------------------
If(
    varHasModuleContext,
    // 2.0) Ensure we have a Module row and Module_ID
    With(
        {
            resolvedModuleId:
                // A) existing ID picked
                If(
                    !IsBlank(varSelectedModuleId),
                    "" & varSelectedModuleId,
                    // B) otherwise create/find by typed name
                    With(
                        {
                            typed: Trim(txtNewModuleName.Text),
                            existingRow: LookUp(Skill_Matrix_Modules, Title = typed)
                        },
                        If(
                            !IsBlank(existingRow),
                            "" & existingRow.Module_ID,
                            // create new module
                            With(
                                { newId: GUID() },
                                Patch(
                                    Skill_Matrix_Modules,
                                    Defaults(Skill_Matrix_Modules),
                                    {
                                        Module_ID: newId,
                                        Title: typed,
                                        Created_By: User().Email,
                                        Created_At: Now()
                                    }
                                );
                                "" & newId
                            )
                        )
                    )
                )
        },
        Set(varModuleIdSafe, resolvedModuleId);
        // Also set name if blank
        If(IsBlank(varSelectedModuleName), Set(varSelectedModuleName, Coalesce(txtNewModuleName.Text, varSelectedModuleName)))
    );

    // 2.1) BEFORE snapshot of Reference for this module
    ClearCollect(
        colRefBeforeSave,
        ShowColumns(
            Filter(Skill_Matrix_Reference, Mod_ID = varModuleIdSafe),
            "Reference_ID", "CatItem_ID", "Mod_ID"
        )
    );

    // 2.2) Build current (before) and target (working) sets
    ClearCollect(
        colBeforeCI,
        ShowColumns(colRefBeforeSave, "CatItem_ID")
    );

    // EXPECTED: colModuleCatItems_Working holds the target CatItem_IDs user wants in the module
    // If your app uses a different working collection, swap it here.
    ClearCollect(
        colTargetCI,
        ShowColumns(colModuleCatItems_Working, "CatItem_ID")
    );

    // 2.3) Compute Adds (in target, not in before) and Removes (in before, not in target)
    ClearCollect(
        colAdds,
        Filter(
            colTargetCI As t,
            CountIf(colBeforeCI, CatItem_ID = t.CatItem_ID) = 0
        )
    );
    ClearCollect(
        colRems,
        Filter(
            colBeforeCI As b,
            CountIf(colTargetCI, CatItem_ID = b.CatItem_ID) = 0
        )
    );

    // 2.4) Apply Adds
    ForAll(
        colAdds As a,
        With(
            {
                ci: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = a.CatItem_ID)
            },
            Patch(
                Skill_Matrix_Reference,
                Defaults(Skill_Matrix_Reference),
                {
                    Reference_ID: GUID(),
                    Mod_ID: varModuleIdSafe,
                    CatItem_ID: a.CatItem_ID,
                    // If you store denormalized labels, fill them here:
                    // Module: varSelectedModuleName,
                    // Category: ci.Category,
                    // Item: ci.Item,
                    Skill_Type: ci.Skill_Type,
                    Created_By: User().Email,
                    Created_At: Now()
                }
            )
        )
    );

    // 2.5) Apply Removes (avoid ForAll-on-same-datasource mutation; use RemoveIf)
    RemoveIf(
        Skill_Matrix_Reference,
        Mod_ID = varModuleIdSafe &&
        CountIf(colRems, CatItem_ID = Skill_Matrix_Reference[@CatItem_ID]) > 0
    );

    // 2.6) AFTER snapshot + diffs for queuing Assignment
    ClearCollect(
        colRefAfterSave,
        ShowColumns(
            Filter(Skill_Matrix_Reference, Mod_ID = varModuleIdSafe),
            "Reference_ID", "CatItem_ID", "Mod_ID"
        )
    );
    ClearCollect(
        colAddedRefs,
        Filter(
            colRefAfterSave As a,
            CountIf(colRefBeforeSave, Reference_ID = a.Reference_ID) = 0
        )
    );
    ClearCollect(
        colRemovedRefs,
        Filter(
            colRefBeforeSave As b,
            CountIf(colRefAfterSave, Reference_ID = b.Reference_ID) = 0
        )
    );

    If(
        CountRows(colAddedRefs) + CountRows(colRemovedRefs) > 0,
        With(
            {
                code: "EditMod-" & Text(Now(), "yyyymmdd-hhnnss"),
                req:  Office365Users.UserProfileV2(User().Email)
            },
            Patch(
                Skill_Matrix_Assignments,
                Defaults(Skill_Matrix_Assignments),
                {
                    Title: code,
                    Operation: { Value: "ModuleUpdated" },
                    Status:    { Value: "Pending" },
                    Module_ID:   varModuleIdSafe,
                    Module_Name: varSelectedModuleName,
                    Added_Reference_IDs:   Concat(colAddedRefs, Reference_ID, ";"),
                    Removed_Reference_IDs: Concat(colRemovedRefs, Reference_ID, ";"),
                    Added_Count:   CountRows(colAddedRefs),
                    Removed_Count: CountRows(colRemovedRefs),
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
    )
);

// ----------------------
// 3) Skill_Type change tracking (independent of module)
// ----------------------
ClearCollect(
    colCIAfter,
    ForAll(
        colCIIds As t,
        {
            CatItem_ID: t.CatItem_ID,
            Skill_Type: LookUp(Skill_Matrix_CategoryItems, CatItem_ID = t.CatItem_ID, Skill_Type)
        }
    )
);

Clear(colCIChanged);
ForAll(
    colCIAfter As a,
    If(
        LookUp(colCIBefore, CatItem_ID = a.CatItem_ID, Skill_Type) <> a.Skill_Type,
        Collect(colCIChanged, { CatItem_ID: a.CatItem_ID })
    )
);
ClearCollect(colCIChanged, Distinct(colCIChanged, CatItem_ID));

If(
    CountRows(colCIChanged) > 0,
    With(
        {
            code: "EditCI-" & Text(Now(), "yyyymmdd-hhnnss"),
            req:  Office365Users.UserProfileV2(User().Email)
        },
        Patch(
            Skill_Matrix_Assignments,
            Defaults(Skill_Matrix_Assignments),
            {
                Title: code,
                Operation: { Value: "CatItem_Update" },
                Status: { Value: "Pending" },
                Updated_CatItem_IDs: Concat(colCIChanged, Text(CatItem_ID) & ";"),
                Updated_Count: CountRows(colCIChanged),
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

// ----------------------
// 4) Final notification
// ----------------------
Set(_modAdds, CountRows(Coalesce(colAddedRefs, Table())));
Set(_modRems, CountRows(Coalesce(colRemovedRefs, Table())));
Set(_ciUpdates, CountRows(Coalesce(colCIChanged, Table())));

If(
    (_modAdds + _modRems + _ciUpdates) = 0,
    Notify("No changes to save.", NotificationType.Information),
    Notify(
        "Saved. Added: " & _modAdds &
        " | Removed: " & _modRems &
        " | Skill-Type updates: " & _ciUpdates,
        NotificationType.Success
    )
);

// Optional post-save refresh/cleanup (uncomment if you want)
// Refresh(Skill_Matrix_Reference);
// Refresh(Skill_Matrix_Assignments);
// Clear(colRefBeforeSave); Clear(colRefAfterSave);
// Clear(colAddedRefs); Clear(colRemovedRefs);
// Clear(colCIUniverse); Clear(colCIIds);
// Clear(colCIBefore); Clear(colCIAfter); Clear(colCIChanged);
