// ------------------------------------------
// CATEGORY-ONLY Skill_Type change detection
// (runs regardless of module context)
// ------------------------------------------

// 1) Build a row-per-visible-item snapshot from the gallery UI
//    Assumes your gallery = galCatItems and each row has:
//      - ThisItem.CatItem_ID
//      - ThisItem.Skill_Type   (current stored type)
//      - a dropdown named ddSkillType with .Selected.Value for the chosen type
ClearCollect(
    colCI_Proposed,
    ForAll(
        galCatItems.AllItems,
        {
            CatItem_ID: ThisItem.CatItem_ID,
            OldType:    ThisItem.Skill_Type,
            NewType:    If(
                            IsRecord(ddSkillType.Selected),
                            ddSkillType.Selected.Value,
                            ddSkillType.Selected
                        )
        }
    )
);

// 2) Keep only rows where the type actually changed
ClearCollect(
    colCI_Changed,
    Filter(colCI_Proposed, !IsBlank(CatItem_ID) && NewType <> OldType)
);

// 3) Persist those changes to CategoryItems
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

// 4) Queue a CatItem_Updated assignment for Flow to fan-out updates
If(
    CountRows(colCI_Changed) > 0,
    With(
        {
            code: "EditCI-" & Text(Now(), "yyyymmdd-hhnnss"),
            req:  Office365Users.UserProfileV2(User().Email)
        },
        Patch(
            Skill_Matrix_Assignments,
            Defaults(Skill_Matrix_Assignments),
            {
                Title:       code,
                Operation:   { Value: "CatItem_Updated" },
                Status:      { Value: "Pending" },
                // semicolon-delimited list of the changed CatItem_IDs
                Updated_CatItem_IDs: Concat(colCI_Changed, CatItem_ID, ";"),
                Updated_Count:       CountRows(colCI_Changed),
                RequestedAt:         Now(),
                RequestedBy: {
                    '@odata.type': "#Microsoft.Azure.Connectors.SharePoint.SPListExpandedUser",
                    Claims: "i:0#.f|membership|" & Coalesce(req.mail, User().Email),
                    DisplayName: Coalesce(req.displayName, User().FullName),
                    Email:       Coalesce(req.mail, User().Email),
                    Department:  Coalesce(req.department, ""),
                    JobTitle:    Coalesce(req.jobTitle, ""),
                    Picture:     ""
                }
            }
        )
    )
);

// 5) Feedback (this runs even if module path also saved things)
If(
    CountRows(colCI_Changed) > 0,
    Notify(
        "Saved " & Text(CountRows(colCI_Changed)) & " item type change(s).",
        NotificationType.Success
    )
);
