
# 今回つまづいた部分
.NET Frameworkでchartを使って折れ線グラフを表示する画面を作成していました。
先輩に確認してもらったところ、グラフの外にY軸（以下Y3軸）を1つ追加してほしいといわれました。下記図でいう一番左のY軸
色々調べた結果、.NET Frameworkだけでなんとかなりました。（2日くらい使った）

# 解決方法
流れをざっくり説明すると、
chartにデータを追加
↓
Y3軸用にSeries型でデータをコピー
↓
Y3軸用データをもとに軸だけを表示する用のchartareaを作成
↓
Y3軸用の表示データを作成（理由は後述）

# コード
```
/// <summary>
/// Y3軸表示用チャートエリア
/// </summary>
private ChartArea areaAxis;

/// <summary>
/// 表示用データ
/// </summary>
private Series scaledSeries;

/// <summary>
/// データコピー用
/// </summary>
private Series seriesCopy;

/// <summary>
/// イニシャライズ
/// </summary>
public DamDataViewer_Graph()
{
    InitializeComponent();
    
    areaAxis = Chart1.ChartAreas.Add("AxisY_");
    scaledSeries = Chart1.Series.Add("_Scaled");
    seriesCopy = Chart1.Series.Add("_Copy");
}

/// <summary>
/// フォーム表示イベント
/// </summary>
/// <param name="sender"></param>
/// <param name="e"></param>
private void Graph_Shown(object sender, EventArgs e)
{
    ChangeChartData();
}

/// <summary>
/// グラフデータ変更処理
/// </summary>
private void ChangeChartData()
{
    var dtNow = DateTime.Now;
    EndTime = dtNow.AddTicks(-(dtNow.Ticks % TimeSpan.TicksPerHour));
    DateTime StartTime = EndTime.AddDays(-10);
    
    for (DateTime dt = StartTime; dt <= EndTime; dt = dt.AddHours(1))
    {
        var val = rand.Next(0, 10) * 0.1;
        Chart1.Series[0].Points.AddXY(dt.ToOADate(), val);
        Chart1.Series[1].Points.AddXY(dt.ToOADate(), val);
        Chart1.Series[2].Points.AddXY(dt.ToOADate(), val);
    }
    CreateYAxis( Chart1.ChartAreas[0], Chart1.Series[7], 15, 10);
    Chart1.Invalidate();
}

/// <summary>
/// 指定したSeriesに対して独立したY軸を作成する
/// </summary>
/// <param name="area">元のChartArea</param>
/// <param name="series">Y軸を追加したいSeries</param>
/// <param name="axisOffset">Y軸の表示位置（左にずらす量）</param>
/// <param name="labelsSize">Y軸ラベルの表示幅</param>
private void CreateYAxis( ChartArea area, Series series, float axisOffset, float labelsSize)
{
    // Seriesのコピーを作成
    seriesCopy.ChartType = series.ChartType;
    foreach (DataPoint point in series.Points)
    {
        seriesCopy.Points.AddXY(point.XValue, point.YValues[0]);
    }

    series.Color = Color.Transparent;
    series.BorderWidth = 0;

    // 軸表示専用のChartAreaを作成（軸だけを表示する）
    areaAxis.BackColor = Color.Transparent;
    areaAxis.BorderColor = Color.Transparent;
    areaAxis.Position.FromRectangleF(area.Position.ToRectangleF());
    areaAxis.InnerPlotPosition.FromRectangleF(area.InnerPlotPosition.ToRectangleF());

    // コピーしたseriesCopyは非表示にする（軸スケール合わせ用）
    seriesCopy.IsVisibleInLegend = false;
    seriesCopy.Color = Color.Transparent;
    seriesCopy.BorderColor = Color.Transparent;
    seriesCopy.ChartArea = areaAxis.Name;

    // 軸表示専用ChartAreaのX軸は非表示にする
    areaAxis.AxisX.LineWidth = 0;
    areaAxis.AxisX.MajorGrid.Enabled = false;
    areaAxis.AxisX.MajorTickMark.Enabled = false;
    areaAxis.AxisX.LabelStyle.Enabled = false;

    // Y軸のグリッドは非表示、スケールは元と同じ
    areaAxis.AxisY.MajorGrid.Enabled = false;
    areaAxis.AxisY.IsStartedFromZero = area.AxisY.IsStartedFromZero;

    // 軸表示用ChartAreaを左にずらして、ラベルスペースを確保
    areaAxis.Position.X -= axisOffset;
    areaAxis.InnerPlotPosition.X += labelsSize;

    // タイトル
    areaAxis.AxisY.Title = "雨量 [m3/s]";
    areaAxis.AxisY.TitleFont = new Font("Microsoft Sans Serif", 12);

    // ラベルフォント
    areaAxis.AxisY.LabelStyle.Font = new Font("Microsoft Sans Serif", 12);

    // 線の太さ
    areaAxis.AxisY.LineWidth = 1;

    ConvertToAxisScale(areaAxis, series, Chart1.ChartAreas[0],false,true);

}

/// <summary>
/// Y軸スケールを統一
/// </summary>
/// <param name="originalChartArea">スケール元チャートエリア</param>
/// <param name="originalSeries">スケール元データ</param>
/// <param name="scaleChartArea">スケール先チャートエリア</param>
/// <param name="originalUseAxisY2">スケール元がY2軸の場合true</param>
/// <param name="scaleUseAxisY2">スケール先がY2軸の場合true</param>
private void ConvertToAxisScale(ChartArea originalChartArea, Series originalSeries, ChartArea scaleChartArea , bool originalUseAxisY2, bool scaleUseAxisY2)
{
    double scaledValue;
    double originalMin = originalUseAxisY2 ? originalChartArea.AxisY2.Minimum : originalChartArea.AxisY.Minimum;
    double originalMax = originalUseAxisY2 ? originalChartArea.AxisY2.Maximum : originalChartArea.AxisY.Maximum;
    double scaleMin = scaleUseAxisY2 ? scaleChartArea.AxisY2.Minimum : scaleChartArea.AxisY.Minimum;
    double scaleMax = scaleUseAxisY2 ? scaleChartArea.AxisY2.Maximum : scaleChartArea.AxisY.Maximum;

    scaledSeries.ChartType = originalSeries.ChartType;
    scaledSeries.YAxisType = originalSeries.YAxisType;
    scaledSeries.Color = Color.Green;
    scaledSeries.LegendText = originalSeries.Name;
    scaledSeries.IsVisibleInLegend = true;
    originalSeries.IsVisibleInLegend = false;
    originalSeries.Color = Color.Transparent;

    // スケール値が取得できない場合
    if (double.IsNaN(originalMin) || double.IsNaN(originalMax) || double.IsNaN(scaleMin) || double.IsNaN(scaleMax))
    {
        return;
    }

    if (originalMax - originalMin < 1)
    {
        return ;
    }

    scaledSeries.Points.Clear();

    // スケール処理
    foreach (DataPoint point in originalSeries.Points)
    {
        scaledValue = scaleMin + (point.YValues[0] - originalMin) * (scaleMax - scaleMin) / (originalMax - originalMin);
        scaledSeries.Points.AddXY(point.XValue, scaledValue);
    }
}
```

# Y3軸用の表示データを作成する理由
表示用データを、Y3軸のChartではなく表示中のChartの目盛りに合わせて描画しているため、
視覚的な整合性を保つために、Y軸スケールを手動で揃える必要があるからです。
（なんていえばいいか難しかったのでコパイロットに考えてもらいました）

# 参考にしたサイト
https://surferonwww.info/BlogEngine/post/2021/11/30/chart-samples-for-windows-forms-application.aspx
https://github.com/geomatics-io/Samples-Environments-for-Microsoft-Chart-Controls

参考にしたコードでは、スケール処理を行う必要がないグラフでしたが、
今回の場合はグラフのX軸をスクロールしたかったのでスケール処理を行っています。