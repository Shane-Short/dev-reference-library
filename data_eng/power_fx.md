With(
    { selType: Coalesce(drpCIType.Selected.Value, drpCIType.SelectedText.Value) },
    RemoveIf(colSkillEdits, CatItem_ID = ThisItem.CatItem_ID);
    Collect(colSkillEdits, { CatItem_ID: ThisItem.CatItem_ID, OldType: ThisItem.Skill_Type, NewType: selType });
    Patch(colModuleCatItems_Working, LookUp(colModuleCatItems_Working, CatItem_ID = ThisItem.CatItem_ID), { StagedType: selType })
)



