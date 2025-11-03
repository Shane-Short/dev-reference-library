// ================== Save Module Config (fast, duplicate-safe) ==================

// 0) Basic user cache (no Office365Users inside loops)
Set(reqUserEmail, User().Email);
Set(reqUserName,  User().FullName);

// 1) Are we editing in module context?
Set(varHasModuleContext, !IsBlank(varSelectedModuleId));

// 2) Build proposed CI table from the row gallery (for Skill_Type sync)
ClearCollect(
    colCI_Proposed,
    AddColumns(
        galCatItems.AllItems,
        NewType, Coalesce(drpCIType.Selected.Value, Skill_Type),
        OldType, Skill_Type
    )
);

// 2.1) Only rows whose Skill_Type changed
ClearCollect(
    colCI_Changed,
    Filter(colCI_Proposed, NewType <> OldType)
);

// 3) Snapshot BEFORE Reference rows for this module (only active)
If(
    varHasModuleContext,
    ClearCollect(
        colRefBeforeSave,
        ShowColumns(
            Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
            Reference_ID, CatItem_ID, Mod_ID
        )
    )
);

// 4) Speed caches (local lookup tables)
Concurrent(
    ClearCollect(
        colCI_ById,
        ShowColumns(Skill_Matrix_CategoryItems, CatItem_ID, Category, Item, Skill_Type)
    ),
    If(
        varHasModuleContext,
        ClearCollect(
            colRef_ByCI,
            ShowColumns(
                Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId),
                Reference_ID, CatItem_ID
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
                ModifiedBy: reqUserEmail,
                ModifiedAt: Now()
            }
        )
    )
);

/* --------- 6) Apply adds/removes for the CURRENT category only --------- */
With(
    { catText: If(IsBlank(cmbSelectCategory.Selected), Blank(), Text(cmbSelectCategory.Selected.Value)) },

    If(
        IsBlank(varSelectedModuleId) || IsBlank(catText),
        Notify("Pick a module and a category.", NotificationType.Warning),

        With(
            {
                /* 6.1 Active references for this module (normalized id to text) */
                curActive:
                    AddColumns(
                        ShowColumns(
                            Filter(
                                Skill_Matrix_Reference,
                                Mod_ID = varSelectedModuleId && IsActive = true
                            ),
                            CatItem_ID
                        ),
                        id, Text(CatItem_ID)
                    ),

                /* 6.2 Desired selection from the working list (only this category) */
                desiredSel:
                    AddColumns(
                        ShowColumns(
                            Filter(colModuleCatItems_Working, IsSelectedForModule = true),
                            CatItem_ID
                        ),
                        id, Text(CatItem_ID)
                    ),

                /* 6.3 All CatItems currently visible (this category) as ids */
                thisCatIds:
                    AddColumns(
                        ShowColumns(
                            Filter(Skill_Matrix_CategoryItems, Text(Category) = catText),
                            CatItem_ID
                        ),
                        id, Text(CatItem_ID)
                    )
            },

            /* 6.4 Compute adds: chosen now but not previously active */
            ClearCollect(
                colAddedRefs,
                Filter(desiredSel As d, CountIf(curActive, id = d.id) = 0)
            );

            /* 6.5 Compute removes: were active in THIS category, now unselected */
            ClearCollect(
                colRemovedRefs,
                Filter(
                    curActive As a,
                    CountIf(thisCatIds, id = a.id) > 0  &&   // only items from this category
                    CountIf(desiredSel, id = a.id) = 0       // now unselected
                )
            );

            /* 6.6 APPLY ADDS (duplicate-safe: reactivate if ref exists, else insert) */
            ForAll(
                colAddedRefs As a,
                With(
                    {
                        ci:     LookUp(colCI_ById, Text(CatItem_ID) = a.id),
                        refRow: LookUp(Skill_Matrix_Reference,
                                       Mod_ID = varSelectedModuleId && Text(CatItem_ID) = a.id)
                    },
                    If(
                        // Re-activate existing ref (even if previously inactive)
                        !IsBlank(refRow),
                        Patch(
                            Skill_Matrix_Reference,
                            refRow,
                            { IsActive: true, UpdatedBy: reqUserEmail, UpdatedAt: Now() }
                        ),
                        // Else create a brand new ref row
                        Patch(
                            Skill_Matrix_Reference,
                            Defaults(Skill_Matrix_Reference),
                            {
                                Reference_ID: GUID(),
                                Mod_ID: varSelectedModuleId,
                                Module: varSelectedModuleName,
                                CatItem_ID: ci.CatItem_ID,
                                Category: ci.Category,
                                Item: ci.Item,
                                /* In your Reference list, the column name is SkillLevel (display "Skill_Type") */
                                SkillLevel: ci.Skill_Type,
                                IsActive: true,
                                Created_By: reqUserEmail,
                                Created_At: Now()
                            }
                        )
                    )
                )
            );

            /* 6.7 APPLY REMOVES (soft delete) */
            ForAll(
                colRemovedRefs As r,
                With(
                    {
                        refRow: LookUp(
                            Skill_Matrix_Reference,
                            Mod_ID = varSelectedModuleId &&
                            Text(CatItem_ID) = r.id
                        )
                    },
                    If(
                        !IsBlank(refRow),
                        Patch(
                            Skill_Matrix_Reference,
                            refRow,
                            { IsActive: false, UpdatedBy: reqUserEmail, UpdatedAt: Now() }
                        )
                    )
                )
            )
        )
    )
);

/* 6.8 Accurate counts + notify */
Set(varAddCount, CountRows(colAddedRefs));
Set(varRemCount, CountRows(colRemovedRefs));
Set(varUpdCount, CountRows(colCI_Changed));

If(
    varAddCount + varRemCount = 0 && varUpdCount = 0,
    Notify("No changes to save.", NotificationType.Information),
    Notify(
        "Saved. Added: " & Text(varAddCount) & " | Removed: " & Text(varRemCount) &
        " | Skill-type updates: " & varUpdCount,
        NotificationType.Success
    )
);

/* 6.9 REBUILD working set so the checkboxes reflect table state immediately */
Refresh(Skill_Matrix_Reference);

// Active refs snapshot (normalize CatItem_ID to text)
ClearCollect(
    colRefActiveIds,
    AddColumns(
        ShowColumns(
            Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
            CatItem_ID
        ),
        id, Text(CatItem_ID)
    )
);

// Category list with IsSelectedForModule precheck
With(
    { catText: If(IsBlank(cmbSelectCategory.Selected), Blank(), Text(cmbSelectCategory.Selected.Value)) },
    If(
        IsBlank(varSelectedModuleId) || IsBlank(catText),
        Clear(colModuleCatItems_Working),
        With(
            {
                catList:
                    AddColumns(
                        Filter(Skill_Matrix_CategoryItems, Text(Category) = catText),
                        id, Text(CatItem_ID)
                    )
            },
            ClearCollect(
                colModuleCatItems_Working,
                AddColumns(
                    catList,
                    IsSelectedForModule,
                    CountIf(colRefActiveIds As r, r.id = id) > 0
                )
            )
        )
    )
);

/* 7) Optional: queue Assignment rows (kept from your existing logic) */
With(
    {
        varCodeNow: "Edit-" & Text(Now(), "yyyymmdd-hhnnss")
    },
    If(
        varHasModuleContext && (varAddCount + varRemCount) > 0,
        With(
            {
                op: If(
                        varAddCount > 0 && varRemCount > 0, "ModCatItemAddRem",
                        If(varAddCount > 0, "ModCatItemAdded", "ModCatItemRemoved")
                    ),
                addedRefsText:   Concat(colAddedRefs,   id, ";"),
                removedRefsText: Concat(colRemovedRefs, id, ";")
            },
            Patch(
                Skill_Matrix_Assignments,
                Defaults(Skill_Matrix_Assignments),
                {
                    Title: varCodeNow,
                    Operation: { Value: op },
                    Status: { Value: "Pending" },

                    Module_ID:  varSelectedModuleId,
                    Module_Name: varSelectedModuleName,

                    Added_Reference_IDs:   addedRefsText,
                    Removed_Reference_IDs: removedRefsText,
                    Added_Count:  varAddCount,
                    Removed_Count: varRemCount,

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

    If(
        varUpdCount > 0,
        Patch(
            Skill_Matrix_Assignments,
            Defaults(Skill_Matrix_Assignments),
            {
                Title: "EditCI-" & Text(Now(), "yyyymmdd-hhnnss"),
                Operation: { Value: "CatItem_Updated" },
                Status: { Value: "Pending" },
                Module_ID:  varSelectedModuleId,
                Module_Name: varSelectedModuleName,
                Updated_CatItem_IDs: Concat(colCI_Changed, CatItem_ID, ";"),
                Updated_Count: varUpdCount,

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

/* 8) Cleanup (avoid mid-save Refresh; one optional refresh at end) */
Clear(colCI_Proposed);
Clear(colCI_Changed);
Clear(colCI_ById);
Clear(colRef_ByCI);
Clear(colRefBeforeSave);
Clear(colAddedRefs);
Clear(colRemovedRefs);
