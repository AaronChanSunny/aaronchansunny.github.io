---
layout: post
title: "Android编码规范（非官方），持续更新中"
date: 2015-11-16 11:30:52 +0800
comments: true
tags: Android
---

这里只是个人习惯的归档，并不是官方规范。包括两大部分：Java代码规范和资源文件命名规范。<!--more-->

## Java Part

### 类成员变量

统一以'm'作为前缀来命名类成员变量。当然有唯一例外，那就是POJO或者JavaBean。

```
public class MainActivity extends AppCompatActivity {
    private static final String TAG = MainActivity.class.getSimpleName();

    private ProgressBarCircularIndeterminate mProgressBar;
    private RecyclerView mRecyclerView;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Toolbar toolbar = (Toolbar) findViewById(R.id.toolbar);
        toolbar.setTitle(R.string.title_main);
        setSupportActionBar(toolbar);

        mProgressBar = (ProgressBarCircularIndeterminate) findViewById(R.id.pb_main);

        mRecyclerView = (RecyclerView) findViewById(R.id.rv_movies);
        mRecyclerView.setHasFixedSize(true);
        mRecyclerView.setLayoutManager(new LinearLayoutManager(this));

        FloatingActionButton fab = (FloatingActionButton) findViewById(R.id.fab);
        fab.setOnClickListener(new View.OnClickListener() {
            @Override
            public void onClick(View view) {
                mProgressBar.setVisibility(View.VISIBLE);
                Movie.doSearch(MainActivity.this, new Movie.SearchListener<Hot>() {
                    @Override
                    public void onResponse(Hot data) {
                        mProgressBar.setVisibility(View.GONE);

                        List<Movie> subjects = data.getSubjects();
                        StringBuilder sb = new StringBuilder();
                        for (Movie item : subjects) {
                            sb.append(item.getTitle()).append("\n");
                        }
                        LogUtil.d(TAG, sb.toString());
                    }
                });
            }
        });
    }

    @Override
    public boolean onCreateOptionsMenu(Menu menu) {
        // Inflate the menu; this adds items to the action bar if it is present.
        getMenuInflater().inflate(R.menu.menu_main, menu);
        return true;
    }

    @Override
    public boolean onOptionsItemSelected(MenuItem item) {
        // Handle action bar item clicks here. The action bar will
        // automatically handle clicks on the Home/Up button, so long
        // as you specify a parent activity in AndroidManifest.xml.
        int id = item.getItemId();

        //noinspection SimplifiableIfStatement
        if (id == R.id.action_settings) {
            return true;
        }

        return super.onOptionsItemSelected(item);
    }
}

public class Rating {
        private float max;
        private float min;
        private float average;
        private String stars;

        public float getMax() {
            return max;
        }

        public void setMax(float max) {
            this.max = max;
        }

        public float getMin() {
            return min;
        }

        public void setMin(float min) {
            this.min = min;
        }

        public float getAverage() {
            return average;
        }

        public void setAverage(float average) {
            this.average = average;
        }

        public String getStars() {
            return stars;
        }

        public void setStars(String stars) {
            this.stars = stars;
        }
    }
```

### 成员变量初始化

若成员变量以'm'前缀，则成员变量初始化不带'this'关键字；若类作为POJO或者JavaBean形式，则仍然以'this'关键字的形式初始化。

## Res Part