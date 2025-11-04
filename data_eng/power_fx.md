Coalesce(
    LookUp(
        colToggleLog,
        CatItem_ID = Text(ThisItem.CatItem_ID)
    ).Desired,
    !IsBlank(
        LookUp(
            colRefActiveIds,
            id = ThisItem.id
        )
    )
)


With({ cid: Text(ThisItem.CatItem_ID) },
    If(
        IsBlank(LookUp(colToggleLog, CatItem_ID = cid)),
        Collect(colToggleLog, { CatItem_ID: cid, Desired: true }),
        Patch(colToggleLog, LookUp(colToggleLog, CatItem_ID = cid), { Desired: true })
    )
)


With({ cid: Text(ThisItem.CatItem_ID) },
    If(
        IsBlank(LookUp(colToggleLog, CatItem_ID = cid)),
        Collect(colToggleLog, { CatItem_ID: cid, Desired: false }),
        Patch(colToggleLog, LookUp(colToggleLog, CatItem_ID = cid), { Desired: false })
    )
)



With(
    {
        cid: Text(ThisItem.CatItem_ID),
        old: ThisItem.Skill_Type,
        new: drpCIType.Selected.Value
    },
    If(
        new <> old,
        If(
            IsBlank(LookUp(colSkillEdits, CatItem_ID = cid)),
            Collect(colSkillEdits, { CatItem_ID: cid, OldType: old, NewType: new }),
            Patch(colSkillEdits, LookUp(colSkillEdits, CatItem_ID = cid), { OldType: old, NewType: new })
        ),
        // reverted back to original -> drop from staged edits
        RemoveIf(colSkillEdits, CatItem_ID = cid)
    )
)



// --- Skill_Type edits (already staged across categories) ---
Set(varUpdCount, CountRows(colSkillEdits));

If(
    varUpdCount > 0,
    Patch(
        Skill_Matrix_Assignments,
        Defaults(Skill_Matrix_Assignments),
        {
            Title: "EditCI-" & Text(Now(), "yyyymmdd-hhnnss"),
            Operation: { Value: "CatItem_Updated" },
            Status: { Value: "Pending" },

            Module_ID:   varSelectedModuleId,
            Module_Name: varSelectedModuleName,

            Updated_CatItem_IDs: Concat(colSkillEdits, CatItem_ID, ";"),
            Updated_Count:       varUpdCount,

            RequestedAt: Now()
        }
    )
);



