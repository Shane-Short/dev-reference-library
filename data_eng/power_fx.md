// ----------------------------------------------
// 0) Basic flags / cleanup
// ----------------------------------------------
Set(varHasModuleContext, !IsBlank(varSelectedModuleId) || !IsBlank(Trim(txtNewModuleName.Text)));

Clear(colDidAdd);
Clear(colDidRemove);
Clear(colRefBeforeSave);
Clear(colRefAfterSave);
Clear(colAddedRefs);
Clear(colRemovedRefs);
Clear(colCI_Proposed);
Clear(colCI_Changed);

// ----------------------------------------------
// 1) MODULE PATH (create/edit module + reference)
// ----------------------------------------------
If(
    varHasModuleContext,

    // 1.1 Resolve Module_ID (use selected, or create/find by typed name)
    With(
        {
            resolvedModuleId:
                If(
                    !IsBlank(varSelectedModuleId),
                    "" & varSelectedModuleId,
                    With(
                        {
                            typedName: Trim(txtNewModuleName.Text),
                            existingMod: LookUp(Skill_Matrix_Modules, Title = typedName)
                        },
                        If(
                            !IsBlank(existingMod),
                            "" & existingMod.Module_ID,
                            With(
                                { newModId: GUID() },
                                Patch(
                                    Skill_Matrix_Modules,
                                    Defaults(Skill_Matrix_Modules),
                                    {
                                        Module_ID: newModId,
                                        Title: typedName,
                                        Created_By: User().Email,
                                        Created_At: Now()
                                    }
                                );
                                "" & newModId
                            )
                        )
                    )
                )
        },
        Set(varModuleIdSafe, resolvedModuleId);
        If(IsBlank(varSelectedModuleName), Set(varSelectedModuleName, Coalesce(txtNewModuleName.Text, varSelectedModuleName)))
    );

    // 1.2 BEFORE snapshot for the module’s Reference rows
    ClearCollect(
        colRefBeforeSave,
        ShowColumns(
            Filter(Skill_Matrix_Reference, Mod_ID = varModuleIdSafe),
            "Reference_ID", "CatItem_ID", "Mod_ID"
        )
    );

    // 1.3 ADD any missing Reference rows for items in colModuleCatItems_Working
    //     (compose a lightweight table of CatItem_IDs we intend to keep)
    With(
        { intended: ShowColumns(colModuleCatItems_Working, "CatItem_ID") },
        ForAll(
            intended As w,
            If(
                IsBlank(
                    LookUp(Skill_Matrix_Reference, Mod_ID = varModuleIdSafe && CatItem_ID = w.CatItem_ID)
                ),
                Patch(
                    Skill_Matrix_Reference,
                    Defaults(Skill_Matrix_Reference),
                    {
                        Reference_ID: GUID(),
                        Mod_ID: varModuleIdSafe,
                        CatItem_ID: w.CatItem_ID,
                        Created_By: User().Email,
                        Created_At: Now()
                    }
                )
            )
        )
    );

    // 1.4 REMOVE any Reference rows no longer in colModuleCatItems_Working
    //     (Avoid ForAll+Remove on same source; use RemoveIf with a predicate)
    RemoveIf(
        Skill_Matrix_Reference,
        Mod_ID = varModuleIdSafe &&
        CountIf(colModuleCatItems_Working, CatItem_ID = Skill_Matrix_Reference[@CatItem_ID]) = 0
    );

    // 1.5 AFTER snapshot
    ClearCollect(
        colRefAfterSave,
        ShowColumns(
            Filter(Skill_Matrix_Reference, Mod_ID = varModuleIdSafe),
            "Reference_ID", "CatItem_ID", "Mod_ID"
        )
    );

    // 1.6 DIFF by Reference_ID (added / removed)
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

    // 1.7 QUEUE appropriate assignment(s) for module edits
    With(
        { req: Office365Users.UserProfileV2(User().Email) },

        // both added and removed
        If(
            CountRows(colAddedRefs) > 0 && CountRows(colRemovedRefs) > 0,
            Patch(
                Skill_Matrix_Assignments,
                Defaults(Skill_Matrix_Assignments),
                {
                    Title: "EditMod-" & Text(Now(), "yyyymmdd-hhnnss"),
                    Operation: { Value: "ModCatItemAddRem" },
                    Status: { Value: "Pending" },
                    Module_ID: varModuleIdSafe,
                    Module_Name: varSelectedModuleName,
                    Added_Reference_ID: Concat(colAddedRefs, Reference_ID, ";"),
                    Removed_Reference_ID: Concat(colRemovedRefs, Reference_ID, ";"),
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
        );

        // only added
        If(
            CountRows(colAddedRefs) > 0 && CountRows(colRemovedRefs) = 0,
            Patch(
                Skill_Matrix_Assignments,
                Defaults(Skill_Matrix_Assignments),
                {
                    Title: "EditMod-" & Text(Now(), "yyyymmdd-hhnnss"),
                    Operation: { Value: "ModCatItemAdded" },
                    Status: { Value: "Pending" },
                    Module_ID: varModuleIdSafe,
                    Module_Name: varSelectedModuleName,
                    Added_Reference_ID: Concat(colAddedRefs, Reference_ID, ";"),
                    Added_Count: CountRows(colAddedRefs),
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
        );

        // only removed
        If(
            CountRows(colRemovedRefs) > 0 && CountRows(colAddedRefs) = 0,
            Patch(
                Skill_Matrix_Assignments,
                Defaults(Skill_Matrix_Assignments),
                {
                    Title: "EditMod-" & Text(Now(), "yyyymmdd-hhnnss"),
                    Operation: { Value: "ModCatItemRemoved" },
                    Status: { Value: "Pending" },
                    Module_ID: varModuleIdSafe,
                    Module_Name: varSelectedModuleName,
                    Removed_Reference_ID: Concat(colRemovedRefs, Reference_ID, ";"),
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

// ----------------------------------------------
// 2) CATEGORY-ONLY Skill_Type edits (no module needed)
//    Also runs when a module IS selected — that’s fine.
// ----------------------------------------------
ClearCollect(
    colCI_Proposed,
    ForAll(
        galCatItems.AllItems,
        {
            CatItem_ID: ThisItem.CatItem_ID,
            OldType: ThisItem.Skill_Type,
            NewType: If(IsRecord(ddSkillType.Selected), ddSkillType.Selected.Value, ddSkillType.Selected)
        }
    )
);

ClearCollect(
    colCI_Changed,
    Filter(colCI_Proposed, !IsBlank(CatItem_ID) && NewType <> OldType)
);

// Persist to CategoryItems (text column Skill_Type)
If(
    CountRows(colCI_Changed) > 0,
    ForAll(
        colCI_Changed As c,
        Patch(
            Skill_Matrix_CategoryItems,
            LookUp(Skill_Matrix_CategoryItems, CatItem_ID = c.CatItem_ID),
            { Skill_Type: c.NewType }
        )
    )
);

// Queue CatItem_Updated (even if a module path also happened)
If(
    CountRows(colCI_Changed) > 0,
    With(
        { req: Office365Users.UserProfileV2(User().Email) },
        Patch(
            Skill_Matrix_Assignments,
            Defaults(Skill_Matrix_Assignments),
            {
                Title: "EditCI-" & Text(Now(), "yyyymmdd-hhnnss"),
                Operation: { Value: "CatItem_Updated" },
                Status: { Value: "Pending" },
                Updated_CatItem_IDs: Concat(colCI_Changed, CatItem_ID, ";"),
                Updated_Count: CountRows(colCI_Changed),
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

// ----------------------------------------------
// 3) FEEDBACK
// ----------------------------------------------
If(
    CountRows(colAddedRefs) + CountRows(colRemovedRefs) + CountRows(colCI_Changed) = 0,
    Notify("No changes to save.", NotificationType.Information),
    Notify(
        "Saved. Added: " & CountRows(colAddedRefs) &
        " | Removed: " & CountRows(colRemovedRefs) &
        " | Type changes: " & CountRows(colCI_Changed),
        NotificationType.Success
    )
);

// ----------------------------------------------
// 4) REFRESH + CLEANUP (optional but recommended)
// ----------------------------------------------
Refresh(Skill_Matrix_Reference);
Refresh(Skill_Matrix_CategoryItems);
Refresh(Skill_Matrix_Assignments);

Clear(colRefBeforeSave);
Clear(colRefAfterSave);
Clear(colAddedRefs);
Clear(colRemovedRefs);
Clear(colCI_Proposed);
Clear(colCI_Changed);

// (optional) Reset inputs
// Reset(txtNewModuleName);
