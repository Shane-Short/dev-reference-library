if(
  equals(length(variables('varPresetIdsForAdds')), 0),
  "Team_Preset_ID eq '' and IsActive eq 1",
  concat(
    "(",
    "Team_Preset_ID eq '",
    first(variables('varPresetIdsForAdds')),
    "'",
    if(
      greater(length(variables('varPresetIdsForAdds')), 1),
      concat(
        " or Team_Preset_ID eq '",
        join(skip(variables('varPresetIdsForAdds'), 1), "' or Team_Preset_ID eq '"),
        "'"
      ),
      ''
    ),
    ") and IsActive eq 1"
  )
)
