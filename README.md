// MainActivity.java
package com.example.appschedule;

import androidx.appcompat.app.AppCompatActivity;
import androidx.appcompat.app.AlertDialog;
import androidx.core.app.ActivityCompat;
import androidx.core.content.ContextCompat;
import android.Manifest;
import android.app.AlarmManager;
import android.app.PendingIntent;
import android.content.Context;
import android.content.Intent;
import android.content.SharedPreferences;
import android.content.res.Configuration;
import android.content.res.Resources;
import android.content.pm.PackageManager;
import android.content.pm.ResolveInfo;
import android.os.Bundle;
import android.util.DisplayMetrics;
import android.widget.ArrayAdapter;
import android.widget.Spinner;
import android.widget.TimePicker;
import android.widget.Toast;
import java.util.Calendar;
import java.util.List;
import java.util.Locale;

public class MainActivity extends AppCompatActivity {
    private TimePicker timePicker;
    private Spinner appSpinner;
    private SharedPreferences sharedPreferences;
    private static final String PREFS_NAME = "AppSchedulePrefs";
    private static final String LANGUAGE_KEY = "language";
    private static final int PERMISSION_REQUEST_CODE = 100;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        sharedPreferences = getSharedPreferences(PREFS_NAME, MODE_PRIVATE);
        String language = sharedPreferences.getString(LANGUAGE_KEY, "en");
        setAppLanguage(language);
        
        setContentView(R.layout.activity_main);

        timePicker = findViewById(R.id.timePicker);
        appSpinner = findViewById(R.id.appSpinner);
        
        checkAndRequestPermissions();
        populateAppSpinner();

        findViewById(R.id.scheduleButton).setOnClickListener(v -> scheduleAppOpening());
        findViewById(R.id.shutdownButton).setOnClickListener(v -> showShutdownConfirmation());
        findViewById(R.id.languageButton).setOnClickListener(v -> showLanguageDialog());
    }

    private void setAppLanguage(String languageCode) {
        Resources res = getResources();
        DisplayMetrics dm = res.getDisplayMetrics();
        Configuration conf = res.getConfiguration();
        conf.setLocale(new Locale(languageCode));
        res.updateConfiguration(conf, dm);
    }

    private void checkAndRequestPermissions() {
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.SET_ALARM) != PackageManager.PERMISSION_GRANTED ||
            ContextCompat.checkSelfPermission(this, Manifest.permission.KILL_BACKGROUND_PROCESSES) != PackageManager.PERMISSION_GRANTED) {
            
            ActivityCompat.requestPermissions(this,
                    new String[]{
                            Manifest.permission.SET_ALARM,
                            Manifest.permission.KILL_BACKGROUND_PROCESSES
                    }, PERMISSION_REQUEST_CODE);
        }
    }

    private void populateAppSpinner() {
        Intent mainIntent = new Intent(Intent.ACTION_MAIN, null);
        mainIntent.addCategory(Intent.CATEGORY_LAUNCHER);
        List<ResolveInfo> appList = getPackageManager().queryIntentActivities(mainIntent, 0);
        String[] appNames = new String[appList.size()];

        for (int i = 0; i < appList.size(); i++) {
            appNames[i] = appList.get(i).activityInfo.applicationInfo.loadLabel(getPackageManager()).toString();
        }

        ArrayAdapter<String> adapter = new ArrayAdapter<>(this,
                android.R.layout.simple_spinner_item, appNames);
        adapter.setDropDownViewResource(android.R.layout.simple_spinner_dropdown_item);
        appSpinner.setAdapter(adapter);
    }

    private void scheduleAppOpening() {
        int hour = timePicker.getHour();
        int minute = timePicker.getMinute();
        String selectedApp = appSpinner.getSelectedItem().toString();

        Calendar calendar = Calendar.getInstance();
        calendar.set(Calendar.HOUR_OF_DAY, hour);
        calendar.set(Calendar.MINUTE, minute);
        calendar.set(Calendar.SECOND, 0);

        if (calendar.getTimeInMillis() <= System.currentTimeMillis()) {
            calendar.add(Calendar.DAY_OF_YEAR, 1);
        }

        Intent intent = new Intent(this, AlarmReceiver.class);
        intent.putExtra("appName", selectedApp);
        PendingIntent pendingIntent = PendingIntent.getBroadcast(this, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_IMMUTABLE);

        AlarmManager alarmManager = (AlarmManager) getSystemService(Context.ALARM_SERVICE);
        alarmManager.setExact(AlarmManager.RTC_WAKEUP, calendar.getTimeInMillis(), pendingIntent);

        Toast.makeText(this, getString(R.string.scheduled_success, selectedApp, hour, minute), Toast.LENGTH_LONG).show();
    }
}

// AlarmReceiver.java
package com.example.appschedule;
import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.content.pm.PackageManager;
import android.content.pm.ResolveInfo;
import android.widget.Toast;
import java.util.List;

public class AlarmReceiver extends BroadcastReceiver {
    @Override
    public void onReceive(Context context, Intent intent) {
        String appName = intent.getStringExtra("appName");
        if (appName == null) return;
        
        try {
            PackageManager pm = context.getPackageManager();
            Intent mainIntent = new Intent(Intent.ACTION_MAIN, null);
            mainIntent.addCategory(Intent.CATEGORY_LAUNCHER);
            List<ResolveInfo> apps = pm.queryIntentActivities(mainIntent, 0);
            
            for (ResolveInfo info : apps) {
                if (appName.equals(info.activityInfo.applicationInfo.loadLabel(pm).toString())) {
                    Intent launchIntent = pm.getLaunchIntentForPackage(info.activityInfo.packageName);
                    if (launchIntent != null) {
                        launchIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
                        context.startActivity(launchIntent);
                        Toast.makeText(context, context.getString(R.string.opening_app, appName), Toast.LENGTH_SHORT).show();
                        return;
                    }
                }
            }
            Toast.makeText(context, context.getString(R.string.app_open_error, appName), Toast.LENGTH_SHORT).show();
        } catch (Exception e) {
            Toast.makeText(context, context.getString(R.string.app_open_error, appName), Toast.LENGTH_SHORT).show();
        }
    }
}
