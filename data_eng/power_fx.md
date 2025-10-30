if(
  equals(length(outputs('SEL_PresetsIds')), 0),
  'Team_Preset_ID eq '''' and IsActive eq 1',
  concat(
    '(',
      join(
        applyToEach(
          items('SEL_PresetsIds'),
          concat('Team_Preset_ID eq ''', item(), '''')
        ),
        ' or '
      ),
    ') and IsActive eq 1'
  )
)
