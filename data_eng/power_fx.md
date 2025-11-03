// ================== Save Module Config (all categories at once) ==================

// 0) Basic user cache (no Office365Users in loops)
Set(reqUserEmail, User().Email);
Set(reqUserName,  User().FullName);

// 1) Validate module context
If(
    IsBlank(varSelectedModuleId),
    Notify("Pick an existing module (or create one) before saving.", NotificationType.Warning);
    Return()
);

// 2) Collect proposed Skill_Type edits from the visible gallery (your current logic)
ClearCollect(
    colCI_Proposed,
    AddColumns(
        galCatItems.AllItems,
        NewType, Coalesce(drpCIType.Selected.Value, Skill_Type),
        OldType, Skill_Type
    )
);
// Only CI whose Skill_Type actually changed
ClearCollect(colCI_Changed, Filter(colCI_Proposed, NewType <> OldType));

// 3) Snapshot CURRENT active Reference rows for this module (normalize CatItem_ID to text once)
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

// 4) Build PENDING selections across ALL categories (from colPendingSel)
ClearCollect(
    colPendTrue,
    AddColumns(Filter(colPendingSel, IsSelected = true),  id, Text(CatItem_ID))
);
ClearCollect(
    colPendFalse,
    AddColumns(Filter(colPendingSel, IsSelected = false), id, Text(CatItem_ID))
);

// 5) Compute DIFFS module-wide
// 5.1 Adds = intended true but not currently active
ClearCollect(
    colAddedRefs,
    Filter(colPendTrue As p, CountIf(colRefActiveIds, id = p.id) = 0)
);

// 5.2 Removes = currently active AND explicitly intended false
ClearCollect(
    colRemovedRefs,
    Filter(colRefActiveIds As a, CountIf(colPendFalse, id = a.id) > 0)
);

// 6) APPLY ADDS (reactivate if found, else create)
//    We pull Category/Item from CategoryItems by CatItem_ID for new rows.
ForAll(
    colAddedRefs As a,
    With(
        {
            ci:      LookUp(Skill_Matrix_CategoryItems, Text(CatItem_ID) = a.id),
            refRow:  LookUp(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && Text(CatItem_ID) = a.id)
        },
        If(
            // Reactivate existing Reference (could be inactive)
            !IsBlank(refRow),
            Patch(
                Skill_Matrix_Reference,
                refRow,
                {
                    IsActive:  true,
                    UpdatedBy: reqUserEmail,
                    UpdatedAt: Now()
                }
            ),
            // Create brand new Reference row
            With(
                { newRefId: GUID() },
                Patch(
                    Skill_Matrix_Reference,
                    Defaults(Skill_Matrix_Reference),
                    {
                        Reference_ID: newRefId,
                        Mod_ID:       varSelectedModuleId,
                        Module:       varSelectedModuleName,
                        CatItem_ID:   ci.CatItem_ID,
                        Category:     ci.Category,
                        Item:         ci.Item,
                        /* In Reference this column is 'SkillLevel' (display 'Skill_Type') */
                        SkillLevel:   ci.Skill_Type,
                        IsActive:     true,
                        Created_By:   reqUserEmail,
                        Created_At:   Now()
                    }
                )
            )
        )
    )
);

// 7) APPLY REMOVES (soft delete)
ForAll(
    colRemovedRefs As r,
    With(
        {
            refRow: LookUp(
                Skill_Matrix_Reference,
                Mod_ID = varSelectedModuleId && Text(CatItem_ID) = r.id
            )
        },
        If(
            !IsBlank(refRow),
            Patch(
                Skill_Matrix_Reference,
                refRow,
                {
                    IsActive:  false,
                    UpdatedBy: reqUserEmail,
                    UpdatedAt: Now()
                }
            )
        )
    )
);

// 8) Persist Skill_Type edits to CategoryItems (your original step)
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
                Modified_At: Now()
            }
        )
    )
);

// 8.1 Also update active Reference.SkillLevel so references stay in sync
If(
    CountRows(colCI_Changed) > 0,
    ForAll(
        colCI_Changed As c,
        ForAll(
            Filter(
                Skill_Matrix_Reference,
                Mod_ID = varSelectedModuleId &&
                CatItem_ID = c.CatItem_ID &&
                IsActive = true
            ) As r,
            Patch(
                Skill_Matrix_Reference,
                r,
                {
                    SkillLevel: c.NewType,
                    UpdatedBy:  reqUserEmail,
                    UpdatedAt:  Now()
                }
            )
        )
    )
);

// 9) Assignment rows (counts + text lists)
Set(varAddCount, CountRows(colAddedRefs));
Set(varRemCount, CountRows(colRemovedRefs));
Set(varUpdCount, CountRows(colCI_Changed));
Set(varCodeNow,  "Edit-" & Text(Now(), "yyyymmdd-hhnnss"));

// 9.1 Module add/remove assignment (only if there were any)
If(
    varAddCount + varRemCount > 0,
    With(
        {
            op: If(varAddCount > 0 && varRemCount > 0, "ModCatItemAddRem", If(varAddCount > 0, "ModCatItemAdded", "ModCatItemRemoved")),
            addedRefsText:   Concat(colAddedRefs,   id, ";"),
            removedRefsText: Concat(colRemovedRefs, id, ";")
        },
        Patch(
            Skill_Matrix_Assignments,
            Defaults(Skill_Matrix_Assignments),
            {
                Title:            varCodeNow,
                Operation:        { Value: op },
                Status:           { Value: "Pending" },
                Module_ID:        varSelectedModuleId,
                Module_Name:      varSelectedModuleName,
                Added_Reference_IDs:   addedRefsText,
                Removed_Reference_IDs: removedRefsText,
                Added_Count:      varAddCount,
                Removed_Count:    varRemCount,
                RequestedAt:      Now(),
                RequestedBy: {
                    '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                    Claims: "i:0#.f|membership| " & reqUserEmail,
                    DisplayName: reqUserName,
                    Email: reqUserEmail
                }
            }
        )
    )
);

// 9.2 Skill-type update assignment
If(
    varUpdCount > 0,
    Patch(
        Skill_Matrix_Assignments,
        Defaults(Skill_Matrix_Assignments),
        {
            Title:       "EditCI-" & Text(Now(), "yyyymmdd-hhnnss"),
            Operation:   { Value: "CatItem_Updated" },
            Status:      { Value: "Pending" },
            Updated_CatItem_IDs: Concat(colCI_Changed, CatItem_ID, ";"),
            Updated_Count:       varUpdCount,
            RequestedAt: Now(),
            RequestedBy: {
                '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                Claims: "i:0#.f|membership| " & reqUserEmail,
                DisplayName: reqUserName,
                Email: reqUserEmail
            }
        }
    )
);

// 10) Notify
If(
    varAddCount + varRemCount + varUpdCount = 0,
    Notify("No changes to save.", NotificationType.Information),
    Notify(
        "Saved. Added: " & Text(varAddCount) &
        " | Removed: " & Text(varRemCount) &
        " | Skill-type updates: " & Text(varUpdCount),
        NotificationType.Success
    )
);

// 11) Refresh + RESEED (keep UI and pending map aligned)
Refresh(Skill_Matrix_Reference);
Refresh(Skill_Matrix_CategoryItems);

// Rebuild active refs list
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

// Reseed pending from truth so navigation starts from the saved state
ClearCollect(
    colPendingSel,
    AddColumns(
        ShowColumns(
            Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
            CatItem_ID
        ),
        IsSelected, true
    )
);

// Rebuild the currently visible category so checkboxes reflect truth immediately
With(
    { catText: If(IsBlank(cmbSelectCategory.Selected), Blank(), Text(cmbSelectCategory.Selected.Value)) },
    If(
        IsBlank(catText),
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
                    With(
                        {
                            baseSel: CountIf(colRefActiveIds As r, r.id = id) > 0,
                            pend:    LookUp(colPendingSel, CatItem_ID = CatItem_ID, IsSelected)
                        },
                        Coalesce(pend, baseSel)
                    )
                )
            )
        )
    )
);

// 12) Clean up temps
Clear(colCI_Proposed); Clear(colCI_Changed);
Clear(colPendTrue);    Clear(colPendFalse);
Clear(colAddedRefs);   Clear(colRemovedRefs);


// After you set varSelectedModuleId & varSelectedModuleName
ClearCollect(
    colPendingSel,
    AddColumns(
        ShowColumns(
            Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true),
            CatItem_ID
        ),
        IsSelected, true
    )
);

// Build colRefActiveIds for this module (used by Defaults below)
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

// Optional: clear the working list until a category is picked
Clear(colModuleCatItems_Working);
Reset(cmbSelectCategory);







// Rebuild visible list – merge base state with pending overrides
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
                    With(
                        {
                            baseSel: CountIf(colRefActiveIds As r, r.id = id) > 0,
                            pend:    LookUp(colPendingSel, CatItem_ID = CatItem_ID, IsSelected)
                        },
                        Coalesce(pend, baseSel)
                    )
                )
            )
        )
    )
);







With(
  {
    baseSel: CountIf(colRefActiveIds, Text(CatItem_ID) = Text(ThisItem.CatItem_ID)) > 0,
    pend:    LookUp(colPendingSel, CatItem_ID = ThisItem.CatItem_ID, IsSelected)
  },
  Coalesce(pend, baseSel)
)






If(
  IsBlank(LookUp(colPendingSel, CatItem_ID = ThisItem.CatItem_ID)),
  Collect(colPendingSel, { CatItem_ID: ThisItem.CatItem_ID, IsSelected: true }),
  UpdateIf(colPendingSel, CatItem_ID = ThisItem.CatItem_ID, { IsSelected: true })
);






If(
  IsBlank(LookUp(colPendingSel, CatItem_ID = ThisItem.CatItem_ID)),
  Collect(colPendingSel, { CatItem_ID: ThisItem.CatItem_ID, IsSelected: false }),
  UpdateIf(colPendingSel, CatItem_ID = ThisItem.CatItem_ID, { IsSelected: false })
);





// seed from current truth so the UI defaults match the list
ClearCollect(colRefActiveIds,
    AddColumns(
        ShowColumns(Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true), CatItem_ID),
        id, Text(CatItem_ID)
    )
);
ClearCollect(colPendingSel,
    AddColumns(
        ShowColumns(Filter(Skill_Matrix_Reference, Mod_ID = varSelectedModuleId && IsActive = true), CatItem_ID),
        IsSelected, true
    )
);
