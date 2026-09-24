# Skill: pb-entry

Generate timesheet entry format untuk Bofis v2.0 Google Sheet — 1 baris TSV lengkap dengan kolom mandatory.

## Format Output

```
PB<number>	<project_name>	<start>	<end>	<pm_status>	<sit_status>	<coverage>	<sonar_status>	<dev>	<tl>	<efficiency>	<planned_md>	<actual_md>	<understand>	<prompting>	<implementation>	<testing>	<debugging>	<manual_dev>	<unit_test>	<documenting>	<ai_tokens>	<ai_model>	<prompt_cycle>	<cost>	<notes>
```

## Rules
- Timesheet breakdown: 8 kolom (menit): understand, prompting, implementation, testing, debugging, manual_dev, unit_test, documenting
- Cost format: $X.XX (USD)
- Notes: ringkasan teknis 1-2 baris — fitur, metrics, bugs, coverage, deploy
- Tanggal: format `YYYY-MM-DD`
- Coverage: `90.0%` format
- Efficiency: `(planned - actual) / planned * 100`
- Model: `deepseek-v4-pro` (default)
