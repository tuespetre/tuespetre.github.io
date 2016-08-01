---
layout: post
title:  "Using Responsive chart.js Charts within React Components"
date:   2015-04-17
categories: react responsive
---

I was trying to use <a href="https://www.npmjs.com/package/react-chartjs">react-chartjs</a>,
which seemed like a good idea at first, but I was having trouble trying to get a chart
to display nicely in a flexbox layout (toolbar at the top, chart stretching to the rest of 
the screen.) I peeked at the source for react-chartjs and I did not like
how/when it was redrawing the underlying chart. It also failed to apply the new height and width
to the underlying chart, causing the chart to always stay the same size. I ended up ditching
the react-chartjs package and using its source to learn how to use chart.js directly. Below is the
result, modified for simplicity and privacy:

{% highlight javascript %}
import React from 'react';
import Chart from 'chart.js';

export default class App extends React.Component {

    constructor(props) {
        super(props);
        
        this.state = { 
            width: '1000', 
            height: '500',
            data: null,
            options: { animation: false }
        };

        this._chart = null; // Doesn't really fit the React lifecycle, so keep it out of state
    }

    render() {
        var width = this.state.width;
        var height = this.state.height;
        var style = { width, height };
        var controls = ...; // omitted

        return (
            <main>
                <div className="controls">
                    {controls}
                </div>
                <div className="chartWrapper" ref="chartWrapper">
                    <canvas ref="canvas" width={width} height={height} style={style} />
                </div>
            </main>
        );
    }

    componentDidMount() {
        // you would load initial data here first

        (window.onresize = () => {
            var wrapper = React.findDOMNode(this.refs.chartWrapper);
            var width = wrapper.clientWidth;
            var height = wrapper.clientHeight;
            
            // this next part is imperative to resizing the chart:
            if (this._chart) {
                this._chart.chart.width = width;
                this._chart.chart.height = height;
            }

            this.setState({ width, height });
        })();
    }

    componentDidUpdate() {
        if (this._chart) this._chart.destroy();

        var ctx = this.refs.canvas.getDOMNode().getContext("2d");
        
        this._chart = new Chart(ctx).Line(this.state.data, this.state.options);
    }

}
{% endhighlight %}

Because the painting of the canvas is sort of 'outside' of the scope of
React and what it does, we don't treat the chart.js object as part of the
React component's state. The canvas itself is most definitely within the
scope of React, though, so its height and width are pertinent to the component's state.
The approach here is to basically let React be React and repaint the canvas
after every update.