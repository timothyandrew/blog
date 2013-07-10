---
layout: post
title: "Using RoboGuice to Inject Views Into a POJO"
date: 2013-07-10 10:50
comments: true
categories: [roboguice, android, DI, java]
---

[RoboGuice](https://github.com/roboguice/roboguice) is great. It lets you get rid of code like:

```java
public class LoginActivity extends Activity {    
    private Button mSignInButtonView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        mSignInButtonView = (Button) findViewById(R.id.sign_in_button);
    }
}
```

Instead, with a single line, RoboGuice will take care of injecting that view into your activity:

```java
public class LoginActivity extends RoboActivity {
    @InjectView(R.id.sign_in_button) Button mSignInButtonView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        // I have a reference to mSignInButtonView here.
    }
}
```

Now we're rewriting [ashoka-survey-mobile](http://github.com/nilenso/ashoka-survey-mobile) as a layered [native app]((http://github.com/nilenso/ashoka-survey-mobile-native)).
We have a LoginView (POJO) which needs references to views present on-screen. Normally, we would instantiate the POJO in the activity, and pass it all the views it needs.

But can we do this with RoboGuice? We can't really use `@InjectView` in a POJO. It needs an Activity context.
The next best thing is to inject the activity into the POJO (RoboGuice is smart enough to inject the correct activity), and pick the views manually using `findViewById`.

So in the activity:

```java
public class LoginActivity extends RoboActivity {
    @Inject LoginView loginView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        loginView.onCreate(); // Need to manually build up the view references inside LoginView
    }
}
```

and in `LoginView`:

```java
public class LoginView {
    @Inject Activity activity; // This gets injected with the correct instance of LoginActivity
    private Button buttonView;
    private EditText editText;
    
    public void onCreate() {
        buttonView = activity.findViewById(R.id.my_button);
        editText = activity.findViewById(R.id.my_edit_text);        
    }
}
```

And then your POJO can manipulate the views that are on-screen as required.