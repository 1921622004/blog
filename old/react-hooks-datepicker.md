## 前言

在最近的项目中，大量的尝试了react hooks，我们的组件库用的是[Next](https://fusion.design)，除了一个地方因为要使用Form + Field的组合，所以使用了class组件，经过了这个项目，也算是总结一些使用的经验，所以准备自己封装一个日历组件来分享一下。以下也会通过git中的commit记录，来分析我的思路。

下来看下效果（因为软件的原因，转的gif竟然如此抖动，顺便求一个mac上转换gif比较好用的软件）


![](https://user-gold-cdn.xitu.io/2019/5/20/16ad5d0f72bf62cb?w=320&h=230&f=png&s=6015)

同时奉上代码地址:  [仓库](https://github.com/1921622004/react-hooks-datepicker)

### 初始化项目
只是使用了create-react-app进行了项目的搭建，然后总体就是hooks + typescript，当然写的不够好，就没有往npm上发的意愿。

### 工具方法，保存状态
首先是写了一个工具方法，作用是将时间戳转化成 年月日 三个属性。 

```js
interface Res {
    year: number;
    month: number;
    day: number;
}

export const getYearMonthDay = (value: number):Res => {
    const date = new Date(value);
    return {
        year: date.getFullYear(),
        month: date.getMonth(),
        day: date.getDay()
    }    
} 
```

然后使用useState，在input的dom上绑定了value，很简单。
```js
const Test:React.FC<TestProps> = memo(({value = Date.now(), onChange}) => {
  console.log("render -------");
  const [date, setDate] = useState<Date>(new Date(value));
  const { year, month, day } = getYearMonthDay(date.getTime());
  console.log(year, month, day);
  return (
    <div>
      <input type="text" value={`${year} - ${month} - ${day}`} />
    </div>
  )
});
```

### 日历遍历的思路
> 这一次的提交主要是确定了日历组件，怎么写，具体思路看下面的代码，尽量通过注释把思路讲的清楚一些。

```js
// 以下两个数组只是为了遍历
const ary7 = new Array(7).fill("");
const ary6 = new Array(6).fill("");

const Test: React.FC<TestProps> = memo(({ value = Date.now(), onChange }) => {
  const [date, setDate] = useState<Date>(new Date(value));
  // useState保存下面弹窗收起/展开的状态
  const [contentVisible, setContentVisible] = useState<boolean>(true);
  const { year, month, day } = getYearMonthDay(date.getTime());
  // 获取当前选中月份的第一天
  const currentMonthFirstDay = new Date(year, month, 1);
  // 判断出这一天是星期几
  const dayOfCurrentMonthFirstDay = currentMonthFirstDay.getDay();
  // 然后当前日历的第一天就应该是月份第一天向前移几天
  const startDay = new Date(currentMonthFirstDay.getTime() - dayOfCurrentMonthFirstDay * 1000 * 60 * 60 * 24);
  const dates:Date[] = [];
  // 生成一个长度为42的数组，里面记录了从第一天开始的每一个date对象
  for (let index = 0; index < 42; index++) {
    dates.push(new Date(startDay.getTime() + 1000 * 60 * 60 * 24 * index));
  }
  return (
    <div>
      <input type="text" value={`${year} - ${month} - ${day}`} />
      {
        contentVisible && (
          <div>
            <div>
              <span>&lt;</span>
              <span>&lt; &lt;</span>
              <span></span>
              <span>&gt;</span>
              <span>&gt; &gt;</span>
            </div>
            <div>
                // 生成的日历应该是7*6的，然后遍历出42个span， 这就是之前生成的两个常量数组的作用
              {
                ary6.map((_, index) => {
                  return (
                    <div>
                      {
                        ary7.map((__, idx) => {
                          const num = index * 7 + idx;
                          console.log(num);
                          
                          const curDate = dates[num]
                          return (
                            <span>{curDate.getDate()}</span>
                          )
                        })
                      }
                    </div>
                  )
                })
              }
            </div>
          </div>
        )
      }
    </div>
  )
});
```

### 处理document点击事件

```js
const Test: React.FC<TestProps> = memo(({ value = Date.now(), onChange }) => {
  // 使用useRef保存最外一层包裹的div
  const wrapper = useRef<HTMLDivElement>(null);
  // 展开收起的方法，都使用了useCallback，传入空数组，让每次渲染都返回相同的函数
  const openContent = useCallback(
    () => setContentVisible(true),
    []
  );
  const closeContent = useCallback(
    () => setContentVisible(false),
    []
  );
  const windowClickhandler = useCallback(
    (ev: MouseEvent) => {
      let target = ev.target as HTMLElement;
      if(wrapper.current && wrapper.current.contains(target)) {
      } else {
        closeContent();
      }
    },
    []
  )
  // 使用useEffect模拟componentDidMount和componentWillUnmount的生命周期函数，来绑定事件
  useEffect(
    () => {
      window.addEventListener("click",windowClickhandler);
      return () => {
        window.removeEventListener('click', windowClickhandler);
      }
    },
    []
  )
  return (
    <div ref={wrapper}>
      // 之前的那些东西，没有变化
    </div>
  )
});
```

### 处理每个日期的点击事件
```js
  // 使用setDate，处理，这里其实第二个参数传递一个空数组即可，因为这个函数是不依赖当前的date状态来变化的。
  const dateClickHandler = useCallback(
    (date:Date) => {
      setDate(date);
      const { year, month, day } = getYearMonthDay(date.getTime());
      onChange(`${year}-${month}-${day}`);
      setContentVisible(false);
    },
    [date]
  )

<span onClick={() => dateClickHandler(curDate)}>{curDate.getDate()}</span>

```

### 支持value传递
```js
// 先判断以下value是否传递，如果没传默认就是当前的时间
const DatePicker: React.FC<TestProps> = ({ value = "", onChange = () => { } }) => {
  let initialDate: Date;
  if (value) {
    let [year, month, day] = value.split('-');
    initialDate = new Date(parseInt(year), parseInt(month) - 1, parseInt(day));
  } else {
    initialDate = new Date();
  }
  const [date, setDate] = useState<Date>(initialDate);
 }
```

### 月份切换的事件处理
> 月份处理就是当前月份的第一天向前移动一个月或者一个月等等。
```js
const DatePicker: React.FC<TestProps> = ({ value = "", onChange = () => { } }) => {
  // 之前这里的currentMonthFirstDay是直接由date得出的，现在成为组件的state就可以让他支持变化。
  const { year, month, day } = getYearMonthDay(date.getTime());
  const [currentMonthFirstDay, setCurrentMonthFirstDay] = useState<Date>(new Date(year, month, 1));
  const { year: chosedYear, month: chosedMonth } = getYearMonthDay(currentMonthFirstDay.getTime());
  const dayOfCurrentMonthFirstDay = currentMonthFirstDay.getDay();
  const startDay = new Date(currentMonthFirstDay.getTime() - dayOfCurrentMonthFirstDay * 1000 * 60 * 60 * 24);
  const dates: Date[] = [];
  for (let index = 0; index < 42; index++) {
    dates.push(new Date(startDay.getTime() + 1000 * 60 * 60 * 24 * index));
  }

  const openContent = useCallback(
    () => setContentVisible(true),
    []
  );
  const closeContent = useCallback(
    () => setContentVisible(false),
    []
  );
  const windowClickhandler = useCallback(
    (ev: MouseEvent) => {
      let target = ev.target as HTMLElement;
      if (wrapper.current && wrapper.current.contains(target)) {
      } else {
        closeContent();
      }
    },
    []
  );
  const dateClickHandler = useCallback(
    (date: Date) => {
      const { year, month, day } = getYearMonthDay(date.getTime());
      onChange(`${year}-${month + 1}-${day}`);
      setContentVisible(false);
    },
    [date]
  );
  // 这里所有的月份切换事件都选择了使用了函数式更新
  const prevMonthHandler = useCallback(
    () => {
      setCurrentMonthFirstDay(value => {
        let { year, month } = getYearMonthDay(value.getTime());
        if (month === 0) {
          month = 11;
          year--;
        } else {
          month--;
        }
        return new Date(year, month, 1)
      })
    },
    []
  );
  const nextMonthHandler = useCallback(
    () => {
      setCurrentMonthFirstDay(value => {
        let { year, month } = getYearMonthDay(value.getTime());
        if (month === 11) {
          month = 0;
          year++;
        } else {
          month++;
        };
        return new Date(year, month, 1);
      })
    },
    []
  );
  const prevYearhandler = useCallback(
    () => {
      setCurrentMonthFirstDay(value => {
        let { year, month } = getYearMonthDay(value.getTime());
        return new Date(--year, month, 1)
      })
    },
    []
  );
  const nextYearHandler = useCallback(
    () => {
      setCurrentMonthFirstDay(value => {
        let { year, month } = getYearMonthDay(value.getTime());
        return new Date(++year, month, 1)
      })
    },
    []
  )
  return (
    <div ref={wrapper}>
      <input type="text" value={`${year} - ${month + 1} - ${day}`} onFocus={openContent} />
      {
        contentVisible && (
          <div className="content">
            <div className="header">
              <span onClick={prevYearhandler}>&lt; &lt;</span>
              <span onClick={prevMonthHandler}>&lt;</span>
              <span>{`${chosedYear} - ${chosedMonth + 1}`}</span>
              <span onClick={nextMonthHandler}>&gt;</span>
              <span onClick={nextYearHandler}>&gt; &gt;</span>
            </div>
          </div>
        )
      }
    </div>
  )
};

```

### 处理porps变化

工具方法
```js
export const getDateFromString = (str: string):Date => {
    let [year, month, day] = str.split('-');
    return new Date(parseInt(year), parseInt(month) - 1, parseInt(day));
}
```

组件
```js
const DatePicker: React.FC<TestProps> = ({ value = "", onChange = () => { } }) => {
  let initialDate: Date;
  if (value) {
    initialDate = getDateFromString(value);
  } else {
    initialDate = new Date();
  };
  
  const [date, setDate] = useState<Date>(initialDate);
  
  // 使用了useRef来保存上一次渲染时传递的value值，
  const prevValue = useRef<string>(value);

  useEffect(
    () => {
      // 仅当value值变化且不同与上一次值时，才会重新进行改变自身date状态。
      if (prevValue.current !== value) {
        let newDate = value ? getDateFromString(value) : new Date();
        setDate(newDate);
        const { year, month } = getYearMonthDay(newDate.getTime());
        setCurrentMonthFirstDay(new Date(year, month, 1))
      }
    },
    [value]
  )

  return (
    ...
  )
};
```
> 这里现在想来其实也可以用useMemo来处理传递进来的value值，这也是一种思路。稍后实现一下。

### 最后
最后贴下代码[github地址](https://github.com/1921622004/react-hooks-datepicker)。