// =============== Save Module Config (fast, robust) ===============

// 0) Basic user cache (no Office365Users in loops)
Set(reqUserEmail, User().Email);
Set(reqUserName,  User().FullName);

// 1) Determine if we’re editing within a module context
Set(
    varHasModuleContext,
    !IsBlank(varSelectedModuleId) || !IsBlank(Trim(txtNewModuleName.Text))
);

// 1.1) Resolve or create Module (Option A)
If(
    varHasModuleContext,
    With(
        {
            typedName: Trim(txtNewModuleName.Text),
            existingMod: If(
                !IsBlank(varSelectedModuleId),
                LookUp(Skill_Matrix_Modules, Module_ID = varSelectedModuleId),
                LookUp(Skill_Matrix_Modules, Title = typedName)
            )
        },
        If(
            !IsBlank(existingMod),
            Set(varModuleIdSafe, "" & existingMod.Module_ID);
            If(IsBlank(varSelectedModuleName), Set(varSelectedModuleName, existingMod.Title)),
            // Create new module
            With(
                { newId: GUID() },
                Patch(
                    Skill_Matrix_Modules,
                    Defaults(Skill_Matrix_Modules),
                    {
                        Module_ID: "" & newId,
                        Title: typedName,
                        Created_By: reqUserEmail,
                        Created_At: Now()
                    }
                );
                Set(varModuleIdSafe, "" & newId);
                Set(varSelectedModuleName, typedName)
            )
        )
    )
);

// 2) Build proposed CI table (from the row gallery) for Skill_Type sync
ClearCollect(
    colCI_Proposed,
    AddColumns(
        galCatItems.AllItems,
        NewType,
            Coalesce(
                drpSkillType.Selected.Value,
                Skill_Type
            ),
        OldType,
            Skill_Type
    )
);

// 2.1) Only the ones whose Skill_Type actually changed
ClearCollect(
    colCI_Changed,
    Filter(colCI_Proposed, NewType <> OldType)
);

// 3) Snapshot BEFORE Reference rows for this module (only if module context)
If(
    varHasModuleContext,
    ClearCollect(
        colRefBeforeSave,
        ShowColumns(
            Filter(Skill_Matrix_Reference, Mod_ID = varModuleIdSafe),
            "Reference_ID", "CatItem_ID", "Mod_ID"
        )
    )
);

// 4) Speed caches (local lookup tables)
Concurrent(
    ClearCollect(
        colCI_ById,
        ShowColumns(Skill_Matrix_CategoryItems, "CatItem_ID", "Category", "Item", "Skill_Type")
    ),
    If(
        varHasModuleContext,
        ClearCollect(
            colRef_ByCI,
            ShowColumns(
                Filter(Skill_Matrix_Reference, Mod_ID = varModuleIdSafe),
                "Reference_ID", "CatItem_ID"
            )
        )
    )
);

// 5) Persist Skill_Type edits to CategoryItems (if any)
If(
    CountRows(colCI_Changed) > 0,
    ForAll(
        colCI_Changed As c,
        Patch(
            Skill_Matrix_CategoryItems,
            LookUp(Skill_Matrix_CategoryItems, CatItem_ID = c.CatItem_ID),
            {
                Skill_Type: c.NewType,
                Modified_By: reqUserEmail,
                Modified_At: Now()
            }
        )
    )
);

// 6) Module add/remove ONLY when in module context
If(
    varHasModuleContext,
    // 6.1) What the user wants the module to contain (target CI IDs)
    ClearCollect(
        colTargetCatItems,
        ShowColumns(colModuleCatItems_Working, "CatItem_ID")
    );

    // 6.2) Add what’s missing (by CatItem_ID not present in BEFORE)
    ClearCollect(
        colRefsToAdd,
        ForAll(
            colTargetCatItems As t,
            If(
                CountIf(colRefBeforeSave, CatItem_ID = t.CatItem_ID) = 0,
                // Prepare payload row with resolved fields
                With(
                    {
                        ci: LookUp(colCI_ById, CatItem_ID = t.CatItem_ID),
                        newRefId: GUID()
                    },
                    Patch(
                        Skill_Matrix_Reference,
                        Defaults(Skill_Matrix_Reference),
                        {
                            Reference_ID: "" & newRefId,
                            Mod_ID: varModuleIdSafe,
                            CatItem_ID: t.CatItem_ID,
                            Category: ci.Category,
                            Item: ci.Item,
                            Skill_Type: ci.Skill_Type,
                            Created_By: reqUserEmail,
                            Created_At: Now()
                        }
                    );
                    // return for collection (captures the new Reference_ID)
                    {
                        Reference_ID: "" & newRefId,
                        CatItem_ID: t.CatItem_ID
                    }
                )
            )
        )
    );

    // 6.3) Remove what should no longer be in the module
    ClearCollect(
        colRefsToRemove,
        ForAll(
            colRefBeforeSave As b,
            If(
                CountIf(colTargetCatItems, CatItem_ID = b.CatItem_ID) = 0,
                // fetch the full record to remove in one call
                LookUp(Skill_Matrix_Reference, Reference_ID = b.Reference_ID)
            )
        )
    );
    If(CountRows(colRefsToRemove) > 0, Remove(Skill_Matrix_Reference, colRefsToRemove));

    // 6.4) AFTER snapshot + diffs (by Reference_ID)
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
    )
);

// 7) Queue Assignment rows (mod adds/removes by Reference_ID; CI skill-type updates by CatItem_ID)
With(
    {
        addCount: If(varHasModuleContext, CountRows(colAddedRefs), 0),
        remCount: If(varHasModuleContext, CountRows(colRemovedRefs), 0),
        updCount: CountRows(colCI_Changed),
        codeNow:  "Edit-" & Text(Now(), "yyyymmdd-hhnnss")
    },
    // 7.1) Module deltas
    If(
        varHasModuleContext && (addCount + remCount) > 0,
        With(
            {
                op:
                    If(
                        addCount > 0 && remCount > 0, "ModCatItemAddRem",
                        If(addCount > 0, "ModCatItemAdded", "ModCatItemRemoved")
                    ),
                addedRefsText:   Concat(colAddedRefs, Reference_ID, ";"),
                removedRefsText: Concat(colRemovedRefs, Reference_ID, ";")
            },
            Patch(
                Skill_Matrix_Assignments,
                Defaults(Skill_Matrix_Assignments),
                {
                    Title: codeNow,
                    Operation: { Value: op },
                    Status: { Value: "Pending" },

                    Module_ID: varModuleIdSafe,
                    Module_Name: varSelectedModuleName,

                    Added_Reference_IDs:   addedRefsText,
                    Removed_Reference_IDs: removedRefsText,
                    Added_Count: addCount,
                    Removed_Count: remCount,

                    RequestedAt: Now(),
                    RequestedBy: {
                        '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                        Claims: "i:0#.f|membership|" & reqUserEmail,
                        DisplayName: reqUserName,
                        Email: reqUserEmail,
                        Department: "",
                        JobTitle: "",
                        Picture: ""
                    }
                }
            )
        )
    );

    // 7.2) CI skill-type updates (always by CatItem_ID, module context optional)
    If(
        updCount > 0,
        Patch(
            Skill_Matrix_Assignments,
            Defaults(Skill_Matrix_Assignments),
            {
                Title: "EditCI-" & Text(Now(), "yyyymmdd-hhnnss"),
                Operation: { Value: "CatItem_Update" },
                Status: { Value: "Pending" },

                // You can optionally include module name/id if present
                Module_ID: If(varHasModuleContext, varModuleIdSafe),
                Module_Name: If(varHasModuleContext, varSelectedModuleName),

                Updated_CatItem_IDs: Concat(colCI_Changed, CatItem_ID, ";"),
                Updated_Count: updCount,

                RequestedAt: Now(),
                RequestedBy: {
                    '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                    Claims: "i:0#.f|membership|" & reqUserEmail,
                    DisplayName: reqUserName,
                    Email: reqUserEmail,
                    Department: "",
                    JobTitle: "",
                    Picture: ""
                }
            }
        )
    );

    // 7.3) User feedback
    If(
        (varHasModuleContext && (addCount + remCount) > 0) || updCount > 0,
        Notify(
            "Saved. Added: " & addCount & " | Removed: " & remCount & " | Skill-type updates: " & updCount,
            NotificationType.Success
        ),
        Notify("No changes to save.", NotificationType.Information)
    )
);

// 8) Cleanup (avoid mid-save Refresh; one optional refresh at end)
Clear(colCI_Proposed);
Clear(colCI_Changed);
Clear(colCI_ById);
Clear(colRef_ByCI);
Clear(colRefBeforeSave);
Clear(colRefAfterSave);
Clear(colAddedRefs);
Clear(colRemovedRefs);
Clear(colRefsToRemove);

// Optional: Refresh lists you show immediately on this screen
// Refresh(Skill_Matrix_Reference);
// Refresh(Skill_Matrix_Assignments);
