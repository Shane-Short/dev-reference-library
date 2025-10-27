// ----------------------
// 0. Validate context
// ----------------------
Set(varHasModuleContext, !IsBlank(varSelectedModuleId) || Len(Trim(txtNewModuleName.Text)) > 0);

// ----------------------
// 1. Build CatItem universe + BEFORE snapshot
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
// 2. Guard: only block module add/remove when no module context
// ----------------------
If(
    !varHasModuleContext && (CountRows(colDidAdd) + CountRows(colDidRemove) > 0),
    Notify(
        "Select an existing Module or enter a new Module name to add or remove items from a module.",
        NotificationType.Error
    ),
    // continue when OK
    // ----------------------------------------------------
    // 3. MODULE-RELATED PATCHES  (only if module context)
    // ----------------------------------------------------
    If(
        varHasModuleContext,
        // 3.1 Ensure Module_ID reference
        Set(
            varModuleIdSafe,
            "" &
            Coalesce(
                If(!IsBlank(varSelectedModuleId), Text(varSelectedModuleId)),
                Text(LookUp(Skill_Matrix_Modules, Title = varSelectedModuleName, Module_ID)),
                Text(FirstN(Skill_Matrix_Modules,1).Module_ID)
            )
        );

        // 3.2 BEFORE snapshot of Reference for this module
        ClearCollect(
            colRefBeforeSave,
            ShowColumns(
                Filter(Skill_Matrix_Reference, Mod_ID = varModuleIdSafe),
                "Reference_ID", "CatItem_ID", "Mod_ID"
            )
        );

        // (your existing logic to patch / update Skill_Matrix_Reference here)
        // This is where you handle adds/removes per module
        // For example: ForAll(colModuleCatItems_Working, Patch(...));

        // 3.3 AFTER snapshot and diffs
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

        // 3.4 Queue EditMod only if changes occurred
        If(
            CountRows(colAddedRefs) + CountRows(colRemovedRefs) > 0,
            With(
                {
                    code: "EditMod-" & Text(Now(), "yyyymmdd-hhnnss"),
                    req: Office365Users.UserProfileV2(User().Email)
                },
                Patch(
                    Skill_Matrix_Assignments,
                    Defaults(Skill_Matrix_Assignments),
                    {
                        Title: code,
                        Operation: { Value: "ModuleUpdated" },
                        Status: { Value: "Pending" },
                        Module_ID: varModuleIdSafe,
                        Module_Name: varSelectedModuleName,
                        Added_Reference_IDs: Concat(colAddedRefs, Reference_ID, ";"),
                        Removed_Reference_IDs: Concat(colRemovedRefs, Reference_ID, ";"),
                        Added_Count: CountRows(colAddedRefs),
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

    // ----------------------------------------------------
    // 4. CATEGORY / ITEM-ONLY  (Skill_Type change tracking)
    // ----------------------------------------------------
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
                req: Office365Users.UserProfileV2(User().Email)
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

    // ----------------------------------------------------
    // 5. Final notification
    // ----------------------------------------------------
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

    // Optional cleanup
    // Refresh(Skill_Matrix_Assignments);
    // Clear(colRefBeforeSave); Clear(colRefAfterSave);
    // Clear(colAddedRefs); Clear(colRemovedRefs);
    // Clear(colCIUniverse); Clear(colCIIds);
    // Clear(colCIBefore); Clear(colCIAfter); Clear(colCIChanged)
);
