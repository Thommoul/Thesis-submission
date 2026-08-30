# Αρχιτεκτονική Κώδικα

Επισκόπηση του τι κάνει στην πράξη το πακέτο `rossmann/` και το `run.py`, module προς module, όπως είναι ο κώδικας σήμερα (όχι όπως περιγράφεται στη διπλωματική — βλ. `04_thesis_review_notes.md` για τα σημεία απόκλισης).

## Ροή δεδομένων

```
data/train.csv, data/test.csv, data/store.csv
        │
        ▼
rossmann/data.py          load_data() / preprocess_data() / load_store_history()
        │
        ├──► rossmann/forecasting.py   Κινητός Μέσος, SES, SARIMAX
        ├──► rossmann/regression.py    OLS regression, Linear Regression ανά κατάστημα/μήνα
        ├──► rossmann/ml.py            Εκπαίδευση XGBoost, backtesting
        ├──► rossmann/analytics.py     αναφορές κλειστών ημερών / μηδενικών πωλήσεων / μέσων πωλήσεων
        └──► rossmann/plots.py         διαγράμματα πυκνότητας πωλήσεων, ρυθμού ανάπτυξης
                │
                ▼
        outputs/*.csv, outputs/*.xlsx, γραφήματα matplotlib
```

Τα `rossmann/features.py` και `rossmann/device.py` είναι κοινές βοηθητικές συναρτήσεις που χρησιμοποιούνται από το `ml.py` (και, ως προς τα features, εννοιολογικά και από το `regression.py` — βλ. σημείωση παρακάτω).

## Αναφορά modules

### `config.py`
Ορίζει τα `BASE_DIR`, `DATA_DIR`, `OUTPUT_DIR`, `MODEL_PATH` και δημιουργεί τον φάκελο `outputs/` αν δεν υπάρχει. Όλα τα υπόλοιπα modules εισάγουν τα paths τους από εδώ — δεν υπάρχει άλλο σημείο όπου τα paths είναι hardcoded.

### `data.py`
- `load_data()` — διαβάζει τα `store.csv`, `test.csv`, `train.csv` όπως είναι.
- `preprocess_data(store, test, train)` — αφαιρεί τις στήλες `StoreType`/`Assortment` από το `store`, αφαιρεί τη στήλη `StateHoliday` από τα `train`/`test`, μετατρέπει το `Date` σε μορφή συμβολοσειράς `dd/mm/yyyy`, και παράγει τις στήλες `Day of the week`, `Month`, `Week/52`. **Σημείωση:** μετά την εκτέλεση αυτής της συνάρτησης, το `Date` είναι πλέον *συμβολοσειρά (string)* και όχι `datetime` — αρκετές επόμενες συναρτήσεις (`forecasting.moving_average`, `analytics.sales_by_day_for_year`) το ξαναμετατρέπουν μόνες τους με `pd.to_datetime`, αντί να βασίζονται σε αυτή την προεπεξεργασία.
- `load_store_history(store_id)` — ανεξάρτητη διαδρομή: διαβάζει ξανά το `train.csv` από την αρχή, το συγχωνεύει με το `store.csv`, φιλτράρει σε ένα κατάστημα και `Open == 1`. Χρησιμοποιείται μόνο από το `ml.py`. Δεν περνάει μέσα από το `preprocess_data`.

### `features.py`
Η μοναδική πηγή αλήθειας για το σύνολο χαρακτηριστικών (features) που χρησιμοποιούν τα μοντέλα μηχανικής μάθησης:
```python
FEATURES = ["Promo", "SchoolHoliday", "CompetitionDistance", "Month",
            "DayOfWeek", "WeekOfYear", "IsWeekend", "IsSummer", "IsChristmas"]
```
Η `add_calendar_features(df)` παράγει τα `Month`, `DayOfWeek`, `WeekOfYear`, τις τρεις δυαδικές (boolean) μεταβλητές, και συμπληρώνει τα κενά του `CompetitionDistance` με τη διάμεσο (median). Χρησιμοποιείται από το `ml.py`. Η `regression.run_rossmann_regression` υλοποιεί εκ νέου, inline, ένα αντίστοιχο (αλλά όχι πανομοιότυπο) block feature engineering αντί να εισάγει αυτό εδώ — βλ. σημειώσεις αναθεώρησης.

### `forecasting.py`
- `moving_average(train, window=7, store_id=None, save=True)` — κυλιόμενος μέσος όρος (rolling mean) του `Sales`, μαζί με τα `Error`, `AbsError`, `SqError`, `APE`. Αποθηκεύει CSV με διαχωριστικό `;`.
- `ses_forecast_store_stats(alpha=0.3, store_id=None)` — `statsmodels.SimpleExpSmoothing` ανά κατάστημα (ή για όλα τα καταστήματα), φιλτραρισμένο σε `Open == 1` και `Sales > 0`. Αποθηκεύει βιβλίο εργασίας Excel με φύλλα `Daily` και `Stats` (MAE/MSE/RMSE/MAPE/Correlation).
- `_fit_sarimax_forecast` / `forecast_sarimax(year, forecast_days)` — `SARIMAX(order=(1,1,1), seasonal_order=(1,1,1,7))` ανά κατάστημα, με πρόβλεψη `forecast_days` ημερών μετά το τέλος του φιλτραρισμένου έτους. Σε περίπτωση αποτυχίας του fit, καταγράφει σιωπηλά τιμές `NaN` για το αντίστοιχο κατάστημα.
- `forecast_store_sales_per_year` / `forecast_store_sales_per_month` — παραλλαγές που είτε εξάγουν και το ιστορικό δίπλα στην πρόβλεψη, είτε απλώς εξάγουν ένα φιλτραρισμένο απόσπασμα πραγματικών τιμών χωρίς κανένα μοντέλο (η "per_month" έκδοση δεν προβλέπει τίποτα — είναι απλώς φιλτραρισμένη εξαγωγή).

### `regression.py`
- `run_rossmann_regression(...)` — συγχωνεύει `train`+`store`, δημιουργεί τα `Year/Month/DayOfWeek/WeekOfYear/IsWeekend/IsSummer/IsChristmas`, εφαρμόζει `statsmodels.OLS` στα `[Month, DayOfWeek, Promo, SchoolHoliday, CompetitionDistance, IsSummer, IsChristmas, IsWeekend]` (διαχωρισμός 80/20), τυπώνει την πλήρη σύνοψη του μοντέλου (συντελεστές, p-values, R²) και το RMSE, με προαιρετική δημιουργία τριών διαγραμμάτων εποχικότητας.
- `predict_sales_filtered(year, month=None)` — πολύ απλούστερο μοντέλο ανά μήνα: `sklearn.LinearRegression` μόνο στα `[DayOfWeek, Promo]`, γράφει ένα φύλλο Excel ανά μήνα με προβλέψεις για κάθε συνδυασμό `(ημέρα, promo)`.
- `predict_sales_for_store(store_id, year=None, month=None)` — το ίδιο απλό μοντέλο δύο χαρακτηριστικών, περιορισμένο σε ένα κατάστημα, γράφει Excel με πραγματικές έναντι προβλεπόμενων τιμών και τυπώνει MAE/RMSE.

### `ml.py`
- `_make_xgb_regressor(**params)` — δημιουργεί `XGBRegressor(tree_method="hist", device=...)`, με τη συσκευή να επιλέγεται από το `device.py`.
- `_train_test_split_by_time(df, features, ...)` — χρονολογικός (όχι τυχαίος) διαχωρισμός 80/20, με στόχο (target) το `log1p(Sales)` εξ ορισμού.
- `run_store_full_analysis(store_id)` — εκπαιδεύει, αξιολογεί, επιστρέφει μοντέλο/αποτελέσματα/RMSE. Δεν αποθηκεύει το μοντέλο μόνιμα.
- `train_xgboost_model(store_id)` — ίδια εκπαίδευση, αλλά **επίσης αποθηκεύει με `joblib.dump` το `(model, None, FEATURES)` στο `MODEL_PATH`** (`rossmann_model.pkl`), το οποίο φορτώνει αργότερα το `backtest_last_30_days`. Είναι η μόνη συνάρτηση που γράφει το αρχείο μοντέλου.
- `xgboost_all_stores_model_detailed()` — ένα ενιαίο global μοντέλο εκπαιδευμένο σε όλα τα καταστήματα (με το `Store` ως αριθμητικό feature), γράφει βιβλίο εργασίας 4 φύλλων (`Global_Predictions`, `Store_Summary`, `Best_Stores`, `Worst_Stores`).
- `backtest_last_30_days(store_id)` — φορτώνει το αποθηκευμένο μοντέλο, το αξιολογεί στις τελευταίες 30 γραμμές πραγματικού ιστορικού του καταστήματος, γράφει Excel dashboard με ενσωματωμένο γράφημα γραμμής μέσω `openpyxl`. **Εξαρτάται από το να έχει τρέξει πρώτα το `train_xgboost_model`** για το συγκεκριμένο κατάστημα — δεν υπάρχει σχετικός έλεγχος, οπότε αν τρέξει το backtest χωρίς προηγούμενη εκπαίδευση, προκύπτει σφάλμα (`FileNotFoundError` ή σφάλμα αποσειριοποίησης).

### `device.py`
`get_xgb_device()` — δοκιμάζει να εκπαιδεύσει έναν `XGBRegressor` 2 γραμμών με `device="cuda"`· επιστρέφει `"cuda"` αν πετύχει, `"cpu"` σε κάθε εξαίρεση. Το αποτέλεσμα αποθηκεύεται σε cache (`lru_cache`) για όλη τη διάρκεια της διεργασίας.

### `analytics.py`
Ανεξάρτητες συναρτήσεις αναφορών, καθεμία διαβάζει το δικό της CSV αντί να δέχεται προ-φορτωμένο dataframe (μόνο οι `sales_by_day_for_year` και `zero_sales_report` δέχονται όρισμα `train`): `sales_by_day_for_year`, `totalsalesofyear`, `check_closed_days`, `average_sales_when_open`, `average_sales_for_store`, `zero_sales_report`.

### `plots.py`
`plot_sales_density()` (KDE πωλήσεων, Promo2 έναντι μη-Promo2) και `plot_sales_growth()` (ημερήσια ποσοστιαία μεταβολή + γραμμή τάσης LOWESS). Και οι δύο καλούν `plt.show()` — προορίζονται για διαδραστική χρήση/notebook, όχι για headless εκτέλεση.

### `run.py` — Σημείο εισόδου CLI
Καταχωρεί βήματα μέσω του decorator `@step("name")` σε ένα dictionary `STEPS`, αυτή τη στιγμή μόνο:
```
moving_average, ses, sales_by_day, sarimax
```
Το `--list` τυπώνει τα καταχωρημένα βήματα, το `--steps a b c` τρέχει ένα επιλεγμένο υποσύνολο, το `--all` τρέχει όλα τα καταχωρημένα βήματα, και χωρίς ορίσματα τρέχει το `moving_average` για το κατάστημα 1. **Μόνο αυτά τα τέσσερα από τις περίπου 15 δημόσιες συναρτήσεις του πακέτου είναι συνδεδεμένα στο CLI** — όλα όσα υπάρχουν στα `regression.py`, `ml.py`, `plots.py`, και τα περισσότερα του `analytics.py` είναι προσβάσιμα αυτή τη στιγμή μόνο με απευθείας import του module σε script/REPL. Βλ. `04_thesis_review_notes.md` για την ασυμφωνία που αυτό δημιουργεί σε σχέση με το `readme.md`.
