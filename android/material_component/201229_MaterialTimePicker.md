# MaterialTimePicker

## 참고

- Android Weekly #446
- [원문 링크](https://blog.stylingandroid.com/materialtimepicker/?utm_source=feedburner&utm_medium=feed&utm_campaign=Feed%3A+StylingAndroid+%28Styling+Android%29)

<br>

## 서론

Material Design Component의 안드로이드 1.3.0-alpha02 버전에서는 `MaterialTimePicker`가 도입되었다. 사용법은 매우 간단하다. 이번 포스트에서 사용법에 대해 다룰것이다. 또한, 글을 작성하고 있는 시점(2020년 12월 중순)에서 한가지 흠이 발견되었다. 이 또한 이야기 해 볼 것이다.

<br>

## 개요

MaterialTimePicker는 사용자에게 시간을 특정(지정)하도록 만들게끔 Dialog를 띄우는 용도로 사용된다. Dialog 자체는 두가지 형태를 가질 수 있다.

1. 시계모양(Mobile Time Picker). 사용자들은 시계를 정면에서 볼 수 있고, `시간 & 분`을 변경할 수 있다.  
또한, `AM & PM`의 선택을 가능하도록 돕는 Chip이 존재한다.

2. 키보드모양(Mobile Time Input). 수정(편집)가능한 시간/분 필드만 보여준다.
마찬가지로, `AM & PM`의 선택이 가능한 Chip이 존재한다.

<img width="200" src="https://user-images.githubusercontent.com/47495592/103269981-5a442880-49fa-11eb-99e1-7d6efe2a6308.jpg">
<img width="200" src="https://user-images.githubusercontent.com/47495592/103269983-5b755580-49fa-11eb-8b0c-f767d4745608.jpg">

사용자들은 두 가지 모드를 변경할 수 있는데, 각 Dialog 좌측 하단에 위치한 버튼을 Tap하여 동작이 이루어진다.



<br>


## 구현

간단한 `Builder Pattern`을 사용하여 이를 구현할 수 있다.

``` kotlin
@AndroidEntryPoint
class TimePickerFragment : Fragment(R.layout.time_picker_fragment) {
 
    private val viewModel: TimePickerViewModel by viewModels()
 
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
 
        TimePickerFragmentBinding.bind(view).also { binding -&gt;
            binding.lifecycleOwner = this
            binding.viewModel = viewModel
            binding.showTimepicker.setOnClickListener {
                val currentTime = viewModel.currentTime()
                MaterialTimePicker.Builder()
                    .setHour(currentTime.hour)
                    .setMinute(currentTime.minute)
                    .setTitleText(R.string.app_name)
                    .setInputMode(viewModel.inputMode.value ?: INPUT_MODE_CLOCK)
                    .build()
                    .apply {
                        addOnPositiveButtonClickListener {
                            viewModel.setSelectedTime(hour, minute)
                        }
                        addOnDismissListener {
                            viewModel.setInputType(inputMode)
                        }
                    }
                    .show(parentFragmentManager, "")
            }
        }
    }
}
```

Builder를 통해, 시간과 분, Input Mode를 설정할 수 있다. 여기서 Input Mode란, Dialog가 보이는 형태를 시계꼴, 혹은 키보드꼴로 할 것인지를 정하는 것이다. Dialog를 Build하고 나서, 여러 다양한 상황에 대한 Listener를 달 수 있다.
(위의 예에서 보이듯, .build() 이후 Listener를 달았음을 확인 가능하다.)

본 코드에서는, viewModel에 현재 선택된 시간이 저장될 수 있도록 Positive Button Click Listener를 설정했다. 또한, 마지막으로 선택되었던 Input Mode를 유지할 수 있도록 Dismiss Listener 설정도 수행했다. 이렇게 함으로써, 사용자가 Dialog를 다시 클릭하게 되었을 경우, 기존에 사용하던 Input Mode대로 Dialog 초기 설정이 이루어진다.

다만, MaterialTimePicker는 Input Mode를 영구적으로 유지시키지는 않는다. 새로운 Instance가 생성될 때 마다 Default Input Type인 시계 모드로 생성된다. 적절한 사용자 경험을 위해서는, 사용자가 Dialog를 Dismiss 할 때의 Input Mode를 영구적으로 저장하는 방법을 고려해야한다. 이로하여금, 사용자의 선호에 따른 일관성있는 동작이 제공될 수 있다.

<br>

## 테마

MaterialTimePicker는 `색상, 타이포그래피, 모양`을 현재의 테마의 것으로부터 가져오도록 설정되어있다. 예를 들어, 내가 앱 테마 색을 전체적으로 초록색 느낌으로 설정했다면, 별다른 코드 수정이 없어도 TimePicker는 초록색 스타일을 가지게 된다.

[공식문서](https://material.io/components/time-pickers)에서 이미 잘 다루고 있기 때문에, 깊이 들어가지는 않는다. 잘 정의 된 테마 설정이 존재한다면, 멋진 느낌의 Component들을 특별한 설정없이 얻을 수 있을 것이다.

<br>


## 문제점

포스팅의 도입부에서, 한가지 문제가 있다고 언급했었다. 이는 MaterialTimePicker의 레이아웃을 깨부수는 이슈였다. TimePicker의 테스트 코드를 모두 작성하고, 이후 모든 의존성(dependency)을 업데이트했다.

<img width="200" src="https://user-images.githubusercontent.com/47495592/103269985-5b755580-49fa-11eb-9497-81771d82fab5.jpg">

그 결과, AM/PM Selector Chip의 위치가 정확하지 않은것을 확인할 수 있었다. 적용했던 다양한 라이브러리들의 버전을 변경해가면서 원인을 찾아보았는데, ConstraintLayout의 최신버전(2.1.0-alpha2)가 이러한 문제를 일으켰음을 확인할 수 있었다. (ConstraintLayout의 버전을 다운그레이드 시키면 정상적으로 표출됨을 확인 가능하다.)

<br>

## 결론

MaterialTimePicker는 잘 디자인되었고, 사용하기 간편한 Component이다. 물론, ConstraintLayout과의 버전 호환성 이슈가 존재하지만 이는 곧 해결 될 문제이다. (버그리포트 제출)