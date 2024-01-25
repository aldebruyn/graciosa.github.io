# graciosa.github.io
<!DOCTYPE html>
<html>
<head>
<meta charset="utf-8" />
<title>leaflet</title>
<script>(function() {
  // If window.HTMLWidgets is already defined, then use it; otherwise create a
  // new object. This allows preceding code to set options that affect the
  // initialization process (though none currently exist).
  window.HTMLWidgets = window.HTMLWidgets || {};

  // See if we're running in a viewer pane. If not, we're in a web browser.
  var viewerMode = window.HTMLWidgets.viewerMode =
      /\bviewer_pane=1\b/.test(window.location);

  // See if we're running in Shiny mode. If not, it's a static document.
  // Note that static widgets can appear in both Shiny and static modes, but
  // obviously, Shiny widgets can only appear in Shiny apps/documents.
  var shinyMode = window.HTMLWidgets.shinyMode =
      typeof(window.Shiny) !== "undefined" && !!window.Shiny.outputBindings;

  // We can't count on jQuery being available, so we implement our own
  // version if necessary.
  function querySelectorAll(scope, selector) {
    if (typeof(jQuery) !== "undefined" && scope instanceof jQuery) {
      return scope.find(selector);
    }
    if (scope.querySelectorAll) {
      return scope.querySelectorAll(selector);
    }
  }

  function asArray(value) {
    if (value === null)
      return [];
    if ($.isArray(value))
      return value;
    return [value];
  }

  // Implement jQuery's extend
  function extend(target /*, ... */) {
    if (arguments.length == 1) {
      return target;
    }
    for (var i = 1; i < arguments.length; i++) {
      var source = arguments[i];
      for (var prop in source) {
        if (source.hasOwnProperty(prop)) {
          target[prop] = source[prop];
        }
      }
    }
    return target;
  }

  // IE8 doesn't support Array.forEach.
  function forEach(values, callback, thisArg) {
    if (values.forEach) {
      values.forEach(callback, thisArg);
    } else {
      for (var i = 0; i < values.length; i++) {
        callback.call(thisArg, values[i], i, values);
      }
    }
  }

  // Replaces the specified method with the return value of funcSource.
  //
  // Note that funcSource should not BE the new method, it should be a function
  // that RETURNS the new method. funcSource receives a single argument that is
  // the overridden method, it can be called from the new method. The overridden
  // method can be called like a regular function, it has the target permanently
  // bound to it so "this" will work correctly.
  function overrideMethod(target, methodName, funcSource) {
    var superFunc = target[methodName] || function() {};
    var superFuncBound = function() {
      return superFunc.apply(target, arguments);
    };
    target[methodName] = funcSource(superFuncBound);
  }

  // Add a method to delegator that, when invoked, calls
  // delegatee.methodName. If there is no such method on
  // the delegatee, but there was one on delegator before
  // delegateMethod was called, then the original version
  // is invoked instead.
  // For example:
  //
  // var a = {
  //   method1: function() { console.log('a1'); }
  //   method2: function() { console.log('a2'); }
  // };
  // var b = {
  //   method1: function() { console.log('b1'); }
  // };
  // delegateMethod(a, b, "method1");
  // delegateMethod(a, b, "method2");
  // a.method1();
  // a.method2();
  //
  // The output would be "b1", "a2".
  function delegateMethod(delegator, delegatee, methodName) {
    var inherited = delegator[methodName];
    delegator[methodName] = function() {
      var target = delegatee;
      var method = delegatee[methodName];

      // The method doesn't exist on the delegatee. Instead,
      // call the method on the delegator, if it exists.
      if (!method) {
        target = delegator;
        method = inherited;
      }

      if (method) {
        return method.apply(target, arguments);
      }
    };
  }

  // Implement a vague facsimilie of jQuery's data method
  function elementData(el, name, value) {
    if (arguments.length == 2) {
      return el["htmlwidget_data_" + name];
    } else if (arguments.length == 3) {
      el["htmlwidget_data_" + name] = value;
      return el;
    } else {
      throw new Error("Wrong number of arguments for elementData: " +
        arguments.length);
    }
  }

  // http://stackoverflow.com/questions/3446170/escape-string-for-use-in-javascript-regex
  function escapeRegExp(str) {
    return str.replace(/[\-\[\]\/\{\}\(\)\*\+\?\.\\\^\$\|]/g, "\\$&");
  }

  function hasClass(el, className) {
    var re = new RegExp("\\b" + escapeRegExp(className) + "\\b");
    return re.test(el.className);
  }

  // elements - array (or array-like object) of HTML elements
  // className - class name to test for
  // include - if true, only return elements with given className;
  //   if false, only return elements *without* given className
  function filterByClass(elements, className, include) {
    var results = [];
    for (var i = 0; i < elements.length; i++) {
      if (hasClass(elements[i], className) == include)
        results.push(elements[i]);
    }
    return results;
  }

  function on(obj, eventName, func) {
    if (obj.addEventListener) {
      obj.addEventListener(eventName, func, false);
    } else if (obj.attachEvent) {
      obj.attachEvent(eventName, func);
    }
  }

  function off(obj, eventName, func) {
    if (obj.removeEventListener)
      obj.removeEventListener(eventName, func, false);
    else if (obj.detachEvent) {
      obj.detachEvent(eventName, func);
    }
  }

  // Translate array of values to top/right/bottom/left, as usual with
  // the "padding" CSS property
  // https://developer.mozilla.org/en-US/docs/Web/CSS/padding
  function unpackPadding(value) {
    if (typeof(value) === "number")
      value = [value];
    if (value.length === 1) {
      return {top: value[0], right: value[0], bottom: value[0], left: value[0]};
    }
    if (value.length === 2) {
      return {top: value[0], right: value[1], bottom: value[0], left: value[1]};
    }
    if (value.length === 3) {
      return {top: value[0], right: value[1], bottom: value[2], left: value[1]};
    }
    if (value.length === 4) {
      return {top: value[0], right: value[1], bottom: value[2], left: value[3]};
    }
  }

  // Convert an unpacked padding object to a CSS value
  function paddingToCss(paddingObj) {
    return paddingObj.top + "px " + paddingObj.right + "px " + paddingObj.bottom + "px " + paddingObj.left + "px";
  }

  // Makes a number suitable for CSS
  function px(x) {
    if (typeof(x) === "number")
      return x + "px";
    else
      return x;
  }

  // Retrieves runtime widget sizing information for an element.
  // The return value is either null, or an object with fill, padding,
  // defaultWidth, defaultHeight fields.
  function sizingPolicy(el) {
    var sizingEl = document.querySelector("script[data-for='" + el.id + "'][type='application/htmlwidget-sizing']");
    if (!sizingEl)
      return null;
    var sp = JSON.parse(sizingEl.textContent || sizingEl.text || "{}");
    if (viewerMode) {
      return sp.viewer;
    } else {
      return sp.browser;
    }
  }

  // @param tasks Array of strings (or falsy value, in which case no-op).
  //   Each element must be a valid JavaScript expression that yields a
  //   function. Or, can be an array of objects with "code" and "data"
  //   properties; in this case, the "code" property should be a string
  //   of JS that's an expr that yields a function, and "data" should be
  //   an object that will be added as an additional argument when that
  //   function is called.
  // @param target The object that will be "this" for each function
  //   execution.
  // @param args Array of arguments to be passed to the functions. (The
  //   same arguments will be passed to all functions.)
  function evalAndRun(tasks, target, args) {
    if (tasks) {
      forEach(tasks, function(task) {
        var theseArgs = args;
        if (typeof(task) === "object") {
          theseArgs = theseArgs.concat([task.data]);
          task = task.code;
        }
        var taskFunc = tryEval(task);
        if (typeof(taskFunc) !== "function") {
          throw new Error("Task must be a function! Source:\n" + task);
        }
        taskFunc.apply(target, theseArgs);
      });
    }
  }

  // Attempt eval() both with and without enclosing in parentheses.
  // Note that enclosing coerces a function declaration into
  // an expression that eval() can parse
  // (otherwise, a SyntaxError is thrown)
  function tryEval(code) {
    var result = null;
    try {
      result = eval("(" + code + ")");
    } catch(error) {
      if (!(error instanceof SyntaxError)) {
        throw error;
      }
      try {
        result = eval(code);
      } catch(e) {
        if (e instanceof SyntaxError) {
          throw error;
        } else {
          throw e;
        }
      }
    }
    return result;
  }

  function initSizing(el) {
    var sizing = sizingPolicy(el);
    if (!sizing)
      return;

    var cel = document.getElementById("htmlwidget_container");
    if (!cel)
      return;

    if (typeof(sizing.padding) !== "undefined") {
      document.body.style.margin = "0";
      document.body.style.padding = paddingToCss(unpackPadding(sizing.padding));
    }

    if (sizing.fill) {
      document.body.style.overflow = "hidden";
      document.body.style.width = "100%";
      document.body.style.height = "100%";
      document.documentElement.style.width = "100%";
      document.documentElement.style.height = "100%";
      cel.style.position = "absolute";
      var pad = unpackPadding(sizing.padding);
      cel.style.top = pad.top + "px";
      cel.style.right = pad.right + "px";
      cel.style.bottom = pad.bottom + "px";
      cel.style.left = pad.left + "px";
      el.style.width = "100%";
      el.style.height = "100%";

      return {
        getWidth: function() { return cel.getBoundingClientRect().width; },
        getHeight: function() { return cel.getBoundingClientRect().height; }
      };

    } else {
      el.style.width = px(sizing.width);
      el.style.height = px(sizing.height);

      return {
        getWidth: function() { return cel.getBoundingClientRect().width; },
        getHeight: function() { return cel.getBoundingClientRect().height; }
      };
    }
  }

  // Default implementations for methods
  var defaults = {
    find: function(scope) {
      return querySelectorAll(scope, "." + this.name);
    },
    renderError: function(el, err) {
      var $el = $(el);

      this.clearError(el);

      // Add all these error classes, as Shiny does
      var errClass = "shiny-output-error";
      if (err.type !== null) {
        // use the classes of the error condition as CSS class names
        errClass = errClass + " " + $.map(asArray(err.type), function(type) {
          return errClass + "-" + type;
        }).join(" ");
      }
      errClass = errClass + " htmlwidgets-error";

      // Is el inline or block? If inline or inline-block, just display:none it
      // and add an inline error.
      var display = $el.css("display");
      $el.data("restore-display-mode", display);

      if (display === "inline" || display === "inline-block") {
        $el.hide();
        if (err.message !== "") {
          var errorSpan = $("<span>").addClass(errClass);
          errorSpan.text(err.message);
          $el.after(errorSpan);
        }
      } else if (display === "block") {
        // If block, add an error just after the el, set visibility:none on the
        // el, and position the error to be on top of the el.
        // Mark it with a unique ID and CSS class so we can remove it later.
        $el.css("visibility", "hidden");
        if (err.message !== "") {
          var errorDiv = $("<div>").addClass(errClass).css("position", "absolute")
            .css("top", el.offsetTop)
            .css("left", el.offsetLeft)
            // setting width can push out the page size, forcing otherwise
            // unnecessary scrollbars to appear and making it impossible for
            // the element to shrink; so use max-width instead
            .css("maxWidth", el.offsetWidth)
            .css("height", el.offsetHeight);
          errorDiv.text(err.message);
          $el.after(errorDiv);

          // Really dumb way to keep the size/position of the error in sync with
          // the parent element as the window is resized or whatever.
          var intId = setInterval(function() {
            if (!errorDiv[0].parentElement) {
              clearInterval(intId);
              return;
            }
            errorDiv
              .css("top", el.offsetTop)
              .css("left", el.offsetLeft)
              .css("maxWidth", el.offsetWidth)
              .css("height", el.offsetHeight);
          }, 500);
        }
      }
    },
    clearError: function(el) {
      var $el = $(el);
      var display = $el.data("restore-display-mode");
      $el.data("restore-display-mode", null);

      if (display === "inline" || display === "inline-block") {
        if (display)
          $el.css("display", display);
        $(el.nextSibling).filter(".htmlwidgets-error").remove();
      } else if (display === "block"){
        $el.css("visibility", "inherit");
        $(el.nextSibling).filter(".htmlwidgets-error").remove();
      }
    },
    sizing: {}
  };

  // Called by widget bindings to register a new type of widget. The definition
  // object can contain the following properties:
  // - name (required) - A string indicating the binding name, which will be
  //   used by default as the CSS classname to look for.
  // - initialize (optional) - A function(el) that will be called once per
  //   widget element; if a value is returned, it will be passed as the third
  //   value to renderValue.
  // - renderValue (required) - A function(el, data, initValue) that will be
  //   called with data. Static contexts will cause this to be called once per
  //   element; Shiny apps will cause this to be called multiple times per
  //   element, as the data changes.
  window.HTMLWidgets.widget = function(definition) {
    if (!definition.name) {
      throw new Error("Widget must have a name");
    }
    if (!definition.type) {
      throw new Error("Widget must have a type");
    }
    // Currently we only support output widgets
    if (definition.type !== "output") {
      throw new Error("Unrecognized widget type '" + definition.type + "'");
    }
    // TODO: Verify that .name is a valid CSS classname

    // Support new-style instance-bound definitions. Old-style class-bound
    // definitions have one widget "object" per widget per type/class of
    // widget; the renderValue and resize methods on such widget objects
    // take el and instance arguments, because the widget object can't
    // store them. New-style instance-bound definitions have one widget
    // object per widget instance; the definition that's passed in doesn't
    // provide renderValue or resize methods at all, just the single method
    //   factory(el, width, height)
    // which returns an object that has renderValue(x) and resize(w, h).
    // This enables a far more natural programming style for the widget
    // author, who can store per-instance state using either OO-style
    // instance fields or functional-style closure variables (I guess this
    // is in contrast to what can only be called C-style pseudo-OO which is
    // what we required before).
    if (definition.factory) {
      definition = createLegacyDefinitionAdapter(definition);
    }

    if (!definition.renderValue) {
      throw new Error("Widget must have a renderValue function");
    }

    // For static rendering (non-Shiny), use a simple widget registration
    // scheme. We also use this scheme for Shiny apps/documents that also
    // contain static widgets.
    window.HTMLWidgets.widgets = window.HTMLWidgets.widgets || [];
    // Merge defaults into the definition; don't mutate the original definition.
    var staticBinding = extend({}, defaults, definition);
    overrideMethod(staticBinding, "find", function(superfunc) {
      return function(scope) {
        var results = superfunc(scope);
        // Filter out Shiny outputs, we only want the static kind
        return filterByClass(results, "html-widget-output", false);
      };
    });
    window.HTMLWidgets.widgets.push(staticBinding);

    if (shinyMode) {
      // Shiny is running. Register the definition with an output binding.
      // The definition itself will not be the output binding, instead
      // we will make an output binding object that delegates to the
      // definition. This is because we foolishly used the same method
      // name (renderValue) for htmlwidgets definition and Shiny bindings
      // but they actually have quite different semantics (the Shiny
      // bindings receive data that includes lots of metadata that it
      // strips off before calling htmlwidgets renderValue). We can't
      // just ignore the difference because in some widgets it's helpful
      // to call this.renderValue() from inside of resize(), and if
      // we're not delegating, then that call will go to the Shiny
      // version instead of the htmlwidgets version.

      // Merge defaults with definition, without mutating either.
      var bindingDef = extend({}, defaults, definition);

      // This object will be our actual Shiny binding.
      var shinyBinding = new Shiny.OutputBinding();

      // With a few exceptions, we'll want to simply use the bindingDef's
      // version of methods if they are available, otherwise fall back to
      // Shiny's defaults. NOTE: If Shiny's output bindings gain additional
      // methods in the future, and we want them to be overrideable by
      // HTMLWidget binding definitions, then we'll need to add them to this
      // list.
      delegateMethod(shinyBinding, bindingDef, "getId");
      delegateMethod(shinyBinding, bindingDef, "onValueChange");
      delegateMethod(shinyBinding, bindingDef, "onValueError");
      delegateMethod(shinyBinding, bindingDef, "renderError");
      delegateMethod(shinyBinding, bindingDef, "clearError");
      delegateMethod(shinyBinding, bindingDef, "showProgress");

      // The find, renderValue, and resize are handled differently, because we
      // want to actually decorate the behavior of the bindingDef methods.

      shinyBinding.find = function(scope) {
        var results = bindingDef.find(scope);

        // Only return elements that are Shiny outputs, not static ones
        var dynamicResults = results.filter(".html-widget-output");

        // It's possible that whatever caused Shiny to think there might be
        // new dynamic outputs, also caused there to be new static outputs.
        // Since there might be lots of different htmlwidgets bindings, we
        // schedule execution for later--no need to staticRender multiple
        // times.
        if (results.length !== dynamicResults.length)
          scheduleStaticRender();

        return dynamicResults;
      };

      // Wrap renderValue to handle initialization, which unfortunately isn't
      // supported natively by Shiny at the time of this writing.

      shinyBinding.renderValue = function(el, data) {
        Shiny.renderDependencies(data.deps);
        // Resolve strings marked as javascript literals to objects
        if (!(data.evals instanceof Array)) data.evals = [data.evals];
        for (var i = 0; data.evals && i < data.evals.length; i++) {
          window.HTMLWidgets.evaluateStringMember(data.x, data.evals[i]);
        }
        if (!bindingDef.renderOnNullValue) {
          if (data.x === null) {
            el.style.visibility = "hidden";
            return;
          } else {
            el.style.visibility = "inherit";
          }
        }
        if (!elementData(el, "initialized")) {
          initSizing(el);

          elementData(el, "initialized", true);
          if (bindingDef.initialize) {
            var rect = el.getBoundingClientRect();
            var result = bindingDef.initialize(el, rect.width, rect.height);
            elementData(el, "init_result", result);
          }
        }
        bindingDef.renderValue(el, data.x, elementData(el, "init_result"));
        evalAndRun(data.jsHooks.render, elementData(el, "init_result"), [el, data.x]);
      };

      // Only override resize if bindingDef implements it
      if (bindingDef.resize) {
        shinyBinding.resize = function(el, width, height) {
          // Shiny can call resize before initialize/renderValue have been
          // called, which doesn't make sense for widgets.
          if (elementData(el, "initialized")) {
            bindingDef.resize(el, width, height, elementData(el, "init_result"));
          }
        };
      }

      Shiny.outputBindings.register(shinyBinding, bindingDef.name);
    }
  };

  var scheduleStaticRenderTimerId = null;
  function scheduleStaticRender() {
    if (!scheduleStaticRenderTimerId) {
      scheduleStaticRenderTimerId = setTimeout(function() {
        scheduleStaticRenderTimerId = null;
        window.HTMLWidgets.staticRender();
      }, 1);
    }
  }

  // Render static widgets after the document finishes loading
  // Statically render all elements that are of this widget's class
  window.HTMLWidgets.staticRender = function() {
    var bindings = window.HTMLWidgets.widgets || [];
    forEach(bindings, function(binding) {
      var matches = binding.find(document.documentElement);
      forEach(matches, function(el) {
        var sizeObj = initSizing(el, binding);

        var getSize = function(el) {
          if (sizeObj) {
            return {w: sizeObj.getWidth(), h: sizeObj.getHeight()}
          } else {
            var rect = el.getBoundingClientRect();
            return {w: rect.width, h: rect.height}
          }
        };

        if (hasClass(el, "html-widget-static-bound"))
          return;
        el.className = el.className + " html-widget-static-bound";

        var initResult;
        if (binding.initialize) {
          var size = getSize(el);
          initResult = binding.initialize(el, size.w, size.h);
          elementData(el, "init_result", initResult);
        }

        if (binding.resize) {
          var lastSize = getSize(el);
          var resizeHandler = function(e) {
            var size = getSize(el);
            if (size.w === 0 && size.h === 0)
              return;
            if (size.w === lastSize.w && size.h === lastSize.h)
              return;
            lastSize = size;
            binding.resize(el, size.w, size.h, initResult);
          };

          on(window, "resize", resizeHandler);

          // This is needed for cases where we're running in a Shiny
          // app, but the widget itself is not a Shiny output, but
          // rather a simple static widget. One example of this is
          // an rmarkdown document that has runtime:shiny and widget
          // that isn't in a render function. Shiny only knows to
          // call resize handlers for Shiny outputs, not for static
          // widgets, so we do it ourselves.
          if (window.jQuery) {
            window.jQuery(document).on(
              "shown.htmlwidgets shown.bs.tab.htmlwidgets shown.bs.collapse.htmlwidgets",
              resizeHandler
            );
            window.jQuery(document).on(
              "hidden.htmlwidgets hidden.bs.tab.htmlwidgets hidden.bs.collapse.htmlwidgets",
              resizeHandler
            );
          }

          // This is needed for the specific case of ioslides, which
          // flips slides between display:none and display:block.
          // Ideally we would not have to have ioslide-specific code
          // here, but rather have ioslides raise a generic event,
          // but the rmarkdown package just went to CRAN so the
          // window to getting that fixed may be long.
          if (window.addEventListener) {
            // It's OK to limit this to window.addEventListener
            // browsers because ioslides itself only supports
            // such browsers.
            on(document, "slideenter", resizeHandler);
            on(document, "slideleave", resizeHandler);
          }
        }

        var scriptData = document.querySelector("script[data-for='" + el.id + "'][type='application/json']");
        if (scriptData) {
          var data = JSON.parse(scriptData.textContent || scriptData.text);
          // Resolve strings marked as javascript literals to objects
          if (!(data.evals instanceof Array)) data.evals = [data.evals];
          for (var k = 0; data.evals && k < data.evals.length; k++) {
            window.HTMLWidgets.evaluateStringMember(data.x, data.evals[k]);
          }
          binding.renderValue(el, data.x, initResult);
          evalAndRun(data.jsHooks.render, initResult, [el, data.x]);
        }
      });
    });

    invokePostRenderHandlers();
  }


  function has_jQuery3() {
    if (!window.jQuery) {
      return false;
    }
    var $version = window.jQuery.fn.jquery;
    var $major_version = parseInt($version.split(".")[0]);
    return $major_version >= 3;
  }

  /*
  / Shiny 1.4 bumped jQuery from 1.x to 3.x which means jQuery's
  / on-ready handler (i.e., $(fn)) is now asyncronous (i.e., it now
  / really means $(setTimeout(fn)).
  / https://jquery.com/upgrade-guide/3.0/#breaking-change-document-ready-handlers-are-now-asynchronous
  /
  / Since Shiny uses $() to schedule initShiny, shiny>=1.4 calls initShiny
  / one tick later than it did before, which means staticRender() is
  / called renderValue() earlier than (advanced) widget authors might be expecting.
  / https://github.com/rstudio/shiny/issues/2630
  /
  / For a concrete example, leaflet has some methods (e.g., updateBounds)
  / which reference Shiny methods registered in initShiny (e.g., setInputValue).
  / Since leaflet is privy to this life-cycle, it knows to use setTimeout() to
  / delay execution of those methods (until Shiny methods are ready)
  / https://github.com/rstudio/leaflet/blob/18ec981/javascript/src/index.js#L266-L268
  /
  / Ideally widget authors wouldn't need to use this setTimeout() hack that
  / leaflet uses to call Shiny methods on a staticRender(). In the long run,
  / the logic initShiny should be broken up so that method registration happens
  / right away, but binding happens later.
  */
  function maybeStaticRenderLater() {
    if (shinyMode && has_jQuery3()) {
      window.jQuery(window.HTMLWidgets.staticRender);
    } else {
      window.HTMLWidgets.staticRender();
    }
  }

  if (document.addEventListener) {
    document.addEventListener("DOMContentLoaded", function() {
      document.removeEventListener("DOMContentLoaded", arguments.callee, false);
      maybeStaticRenderLater();
    }, false);
  } else if (document.attachEvent) {
    document.attachEvent("onreadystatechange", function() {
      if (document.readyState === "complete") {
        document.detachEvent("onreadystatechange", arguments.callee);
        maybeStaticRenderLater();
      }
    });
  }


  window.HTMLWidgets.getAttachmentUrl = function(depname, key) {
    // If no key, default to the first item
    if (typeof(key) === "undefined")
      key = 1;

    var link = document.getElementById(depname + "-" + key + "-attachment");
    if (!link) {
      throw new Error("Attachment " + depname + "/" + key + " not found in document");
    }
    return link.getAttribute("href");
  };

  window.HTMLWidgets.dataframeToD3 = function(df) {
    var names = [];
    var length;
    for (var name in df) {
        if (df.hasOwnProperty(name))
            names.push(name);
        if (typeof(df[name]) !== "object" || typeof(df[name].length) === "undefined") {
            throw new Error("All fields must be arrays");
        } else if (typeof(length) !== "undefined" && length !== df[name].length) {
            throw new Error("All fields must be arrays of the same length");
        }
        length = df[name].length;
    }
    var results = [];
    var item;
    for (var row = 0; row < length; row++) {
        item = {};
        for (var col = 0; col < names.length; col++) {
            item[names[col]] = df[names[col]][row];
        }
        results.push(item);
    }
    return results;
  };

  window.HTMLWidgets.transposeArray2D = function(array) {
      if (array.length === 0) return array;
      var newArray = array[0].map(function(col, i) {
          return array.map(function(row) {
              return row[i]
          })
      });
      return newArray;
  };
  // Split value at splitChar, but allow splitChar to be escaped
  // using escapeChar. Any other characters escaped by escapeChar
  // will be included as usual (including escapeChar itself).
  function splitWithEscape(value, splitChar, escapeChar) {
    var results = [];
    var escapeMode = false;
    var currentResult = "";
    for (var pos = 0; pos < value.length; pos++) {
      if (!escapeMode) {
        if (value[pos] === splitChar) {
          results.push(currentResult);
          currentResult = "";
        } else if (value[pos] === escapeChar) {
          escapeMode = true;
        } else {
          currentResult += value[pos];
        }
      } else {
        currentResult += value[pos];
        escapeMode = false;
      }
    }
    if (currentResult !== "") {
      results.push(currentResult);
    }
    return results;
  }
  // Function authored by Yihui/JJ Allaire
  window.HTMLWidgets.evaluateStringMember = function(o, member) {
    var parts = splitWithEscape(member, '.', '\\');
    for (var i = 0, l = parts.length; i < l; i++) {
      var part = parts[i];
      // part may be a character or 'numeric' member name
      if (o !== null && typeof o === "object" && part in o) {
        if (i == (l - 1)) { // if we are at the end of the line then evalulate
          if (typeof o[part] === "string")
            o[part] = tryEval(o[part]);
        } else { // otherwise continue to next embedded object
          o = o[part];
        }
      }
    }
  };

  // Retrieve the HTMLWidget instance (i.e. the return value of an
  // HTMLWidget binding's initialize() or factory() function)
  // associated with an element, or null if none.
  window.HTMLWidgets.getInstance = function(el) {
    return elementData(el, "init_result");
  };

  // Finds the first element in the scope that matches the selector,
  // and returns the HTMLWidget instance (i.e. the return value of
  // an HTMLWidget binding's initialize() or factory() function)
  // associated with that element, if any. If no element matches the
  // selector, or the first matching element has no HTMLWidget
  // instance associated with it, then null is returned.
  //
  // The scope argument is optional, and defaults to window.document.
  window.HTMLWidgets.find = function(scope, selector) {
    if (arguments.length == 1) {
      selector = scope;
      scope = document;
    }

    var el = scope.querySelector(selector);
    if (el === null) {
      return null;
    } else {
      return window.HTMLWidgets.getInstance(el);
    }
  };

  // Finds all elements in the scope that match the selector, and
  // returns the HTMLWidget instances (i.e. the return values of
  // an HTMLWidget binding's initialize() or factory() function)
  // associated with the elements, in an array. If elements that
  // match the selector don't have an associated HTMLWidget
  // instance, the returned array will contain nulls.
  //
  // The scope argument is optional, and defaults to window.document.
  window.HTMLWidgets.findAll = function(scope, selector) {
    if (arguments.length == 1) {
      selector = scope;
      scope = document;
    }

    var nodes = scope.querySelectorAll(selector);
    var results = [];
    for (var i = 0; i < nodes.length; i++) {
      results.push(window.HTMLWidgets.getInstance(nodes[i]));
    }
    return results;
  };

  var postRenderHandlers = [];
  function invokePostRenderHandlers() {
    while (postRenderHandlers.length) {
      var handler = postRenderHandlers.shift();
      if (handler) {
        handler();
      }
    }
  }

  // Register the given callback function to be invoked after the
  // next time static widgets are rendered.
  window.HTMLWidgets.addPostRenderHandler = function(callback) {
    postRenderHandlers.push(callback);
  };

  // Takes a new-style instance-bound definition, and returns an
  // old-style class-bound definition. This saves us from having
  // to rewrite all the logic in this file to accomodate both
  // types of definitions.
  function createLegacyDefinitionAdapter(defn) {
    var result = {
      name: defn.name,
      type: defn.type,
      initialize: function(el, width, height) {
        return defn.factory(el, width, height);
      },
      renderValue: function(el, x, instance) {
        return instance.renderValue(x);
      },
      resize: function(el, width, height, instance) {
        return instance.resize(width, height);
      }
    };

    if (defn.find)
      result.find = defn.find;
    if (defn.renderError)
      result.renderError = defn.renderError;
    if (defn.clearError)
      result.clearError = defn.clearError;

    return result;
  }
})();
</script>
<script>/*! jQuery v3.6.0 | (c) OpenJS Foundation and other contributors | jquery.org/license */
!function(e,t){"use strict";"object"==typeof module&&"object"==typeof module.exports?module.exports=e.document?t(e,!0):function(e){if(!e.document)throw new Error("jQuery requires a window with a document");return t(e)}:t(e)}("undefined"!=typeof window?window:this,function(C,e){"use strict";var t=[],r=Object.getPrototypeOf,s=t.slice,g=t.flat?function(e){return t.flat.call(e)}:function(e){return t.concat.apply([],e)},u=t.push,i=t.indexOf,n={},o=n.toString,v=n.hasOwnProperty,a=v.toString,l=a.call(Object),y={},m=function(e){return"function"==typeof e&&"number"!=typeof e.nodeType&&"function"!=typeof e.item},x=function(e){return null!=e&&e===e.window},E=C.document,c={type:!0,src:!0,nonce:!0,noModule:!0};function b(e,t,n){var r,i,o=(n=n||E).createElement("script");if(o.text=e,t)for(r in c)(i=t[r]||t.getAttribute&&t.getAttribute(r))&&o.setAttribute(r,i);n.head.appendChild(o).parentNode.removeChild(o)}function w(e){return null==e?e+"":"object"==typeof e||"function"==typeof e?n[o.call(e)]||"object":typeof e}var f="3.6.0",S=function(e,t){return new S.fn.init(e,t)};function p(e){var t=!!e&&"length"in e&&e.length,n=w(e);return!m(e)&&!x(e)&&("array"===n||0===t||"number"==typeof t&&0<t&&t-1 in e)}S.fn=S.prototype={jquery:f,constructor:S,length:0,toArray:function(){return s.call(this)},get:function(e){return null==e?s.call(this):e<0?this[e+this.length]:this[e]},pushStack:function(e){var t=S.merge(this.constructor(),e);return t.prevObject=this,t},each:function(e){return S.each(this,e)},map:function(n){return this.pushStack(S.map(this,function(e,t){return n.call(e,t,e)}))},slice:function(){return this.pushStack(s.apply(this,arguments))},first:function(){return this.eq(0)},last:function(){return this.eq(-1)},even:function(){return this.pushStack(S.grep(this,function(e,t){return(t+1)%2}))},odd:function(){return this.pushStack(S.grep(this,function(e,t){return t%2}))},eq:function(e){var t=this.length,n=+e+(e<0?t:0);return this.pushStack(0<=n&&n<t?[this[n]]:[])},end:function(){return this.prevObject||this.constructor()},push:u,sort:t.sort,splice:t.splice},S.extend=S.fn.extend=function(){var e,t,n,r,i,o,a=arguments[0]||{},s=1,u=arguments.length,l=!1;for("boolean"==typeof a&&(l=a,a=arguments[s]||{},s++),"object"==typeof a||m(a)||(a={}),s===u&&(a=this,s--);s<u;s++)if(null!=(e=arguments[s]))for(t in e)r=e[t],"__proto__"!==t&&a!==r&&(l&&r&&(S.isPlainObject(r)||(i=Array.isArray(r)))?(n=a[t],o=i&&!Array.isArray(n)?[]:i||S.isPlainObject(n)?n:{},i=!1,a[t]=S.extend(l,o,r)):void 0!==r&&(a[t]=r));return a},S.extend({expando:"jQuery"+(f+Math.random()).replace(/\D/g,""),isReady:!0,error:function(e){throw new Error(e)},noop:function(){},isPlainObject:function(e){var t,n;return!(!e||"[object Object]"!==o.call(e))&&(!(t=r(e))||"function"==typeof(n=v.call(t,"constructor")&&t.constructor)&&a.call(n)===l)},isEmptyObject:function(e){var t;for(t in e)return!1;return!0},globalEval:function(e,t,n){b(e,{nonce:t&&t.nonce},n)},each:function(e,t){var n,r=0;if(p(e)){for(n=e.length;r<n;r++)if(!1===t.call(e[r],r,e[r]))break}else for(r in e)if(!1===t.call(e[r],r,e[r]))break;return e},makeArray:function(e,t){var n=t||[];return null!=e&&(p(Object(e))?S.merge(n,"string"==typeof e?[e]:e):u.call(n,e)),n},inArray:function(e,t,n){return null==t?-1:i.call(t,e,n)},merge:function(e,t){for(var n=+t.length,r=0,i=e.length;r<n;r++)e[i++]=t[r];return e.length=i,e},grep:function(e,t,n){for(var r=[],i=0,o=e.length,a=!n;i<o;i++)!t(e[i],i)!==a&&r.push(e[i]);return r},map:function(e,t,n){var r,i,o=0,a=[];if(p(e))for(r=e.length;o<r;o++)null!=(i=t(e[o],o,n))&&a.push(i);else for(o in e)null!=(i=t(e[o],o,n))&&a.push(i);return g(a)},guid:1,support:y}),"function"==typeof Symbol&&(S.fn[Symbol.iterator]=t[Symbol.iterator]),S.each("Boolean Number String Function Array Date RegExp Object Error Symbol".split(" "),function(e,t){n["[object "+t+"]"]=t.toLowerCase()});var d=function(n){var e,d,b,o,i,h,f,g,w,u,l,T,C,a,E,v,s,c,y,S="sizzle"+1*new Date,p=n.document,k=0,r=0,m=ue(),x=ue(),A=ue(),N=ue(),j=function(e,t){return e===t&&(l=!0),0},D={}.hasOwnProperty,t=[],q=t.pop,L=t.push,H=t.push,O=t.slice,P=function(e,t){for(var n=0,r=e.length;n<r;n++)if(e[n]===t)return n;return-1},R="checked|selected|async|autofocus|autoplay|controls|defer|disabled|hidden|ismap|loop|multiple|open|readonly|required|scoped",M="[\\x20\\t\\r\\n\\f]",I="(?:\\\\[\\da-fA-F]{1,6}"+M+"?|\\\\[^\\r\\n\\f]|[\\w-]|[^\0-\\x7f])+",W="\\["+M+"*("+I+")(?:"+M+"*([*^$|!~]?=)"+M+"*(?:'((?:\\\\.|[^\\\\'])*)'|\"((?:\\\\.|[^\\\\\"])*)\"|("+I+"))|)"+M+"*\\]",F=":("+I+")(?:\\((('((?:\\\\.|[^\\\\'])*)'|\"((?:\\\\.|[^\\\\\"])*)\")|((?:\\\\.|[^\\\\()[\\]]|"+W+")*)|.*)\\)|)",B=new RegExp(M+"+","g"),$=new RegExp("^"+M+"+|((?:^|[^\\\\])(?:\\\\.)*)"+M+"+$","g"),_=new RegExp("^"+M+"*,"+M+"*"),z=new RegExp("^"+M+"*([>+~]|"+M+")"+M+"*"),U=new RegExp(M+"|>"),X=new RegExp(F),V=new RegExp("^"+I+"$"),G={ID:new RegExp("^#("+I+")"),CLASS:new RegExp("^\\.("+I+")"),TAG:new RegExp("^("+I+"|[*])"),ATTR:new RegExp("^"+W),PSEUDO:new RegExp("^"+F),CHILD:new RegExp("^:(only|first|last|nth|nth-last)-(child|of-type)(?:\\("+M+"*(even|odd|(([+-]|)(\\d*)n|)"+M+"*(?:([+-]|)"+M+"*(\\d+)|))"+M+"*\\)|)","i"),bool:new RegExp("^(?:"+R+")$","i"),needsContext:new RegExp("^"+M+"*[>+~]|:(even|odd|eq|gt|lt|nth|first|last)(?:\\("+M+"*((?:-\\d)?\\d*)"+M+"*\\)|)(?=[^-]|$)","i")},Y=/HTML$/i,Q=/^(?:input|select|textarea|button)$/i,J=/^h\d$/i,K=/^[^{]+\{\s*\[native \w/,Z=/^(?:#([\w-]+)|(\w+)|\.([\w-]+))$/,ee=/[+~]/,te=new RegExp("\\\\[\\da-fA-F]{1,6}"+M+"?|\\\\([^\\r\\n\\f])","g"),ne=function(e,t){var n="0x"+e.slice(1)-65536;return t||(n<0?String.fromCharCode(n+65536):String.fromCharCode(n>>10|55296,1023&n|56320))},re=/([\0-\x1f\x7f]|^-?\d)|^-$|[^\0-\x1f\x7f-\uFFFF\w-]/g,ie=function(e,t){return t?"\0"===e?"\ufffd":e.slice(0,-1)+"\\"+e.charCodeAt(e.length-1).toString(16)+" ":"\\"+e},oe=function(){T()},ae=be(function(e){return!0===e.disabled&&"fieldset"===e.nodeName.toLowerCase()},{dir:"parentNode",next:"legend"});try{H.apply(t=O.call(p.childNodes),p.childNodes),t[p.childNodes.length].nodeType}catch(e){H={apply:t.length?function(e,t){L.apply(e,O.call(t))}:function(e,t){var n=e.length,r=0;while(e[n++]=t[r++]);e.length=n-1}}}function se(t,e,n,r){var i,o,a,s,u,l,c,f=e&&e.ownerDocument,p=e?e.nodeType:9;if(n=n||[],"string"!=typeof t||!t||1!==p&&9!==p&&11!==p)return n;if(!r&&(T(e),e=e||C,E)){if(11!==p&&(u=Z.exec(t)))if(i=u[1]){if(9===p){if(!(a=e.getElementById(i)))return n;if(a.id===i)return n.push(a),n}else if(f&&(a=f.getElementById(i))&&y(e,a)&&a.id===i)return n.push(a),n}else{if(u[2])return H.apply(n,e.getElementsByTagName(t)),n;if((i=u[3])&&d.getElementsByClassName&&e.getElementsByClassName)return H.apply(n,e.getElementsByClassName(i)),n}if(d.qsa&&!N[t+" "]&&(!v||!v.test(t))&&(1!==p||"object"!==e.nodeName.toLowerCase())){if(c=t,f=e,1===p&&(U.test(t)||z.test(t))){(f=ee.test(t)&&ye(e.parentNode)||e)===e&&d.scope||((s=e.getAttribute("id"))?s=s.replace(re,ie):e.setAttribute("id",s=S)),o=(l=h(t)).length;while(o--)l[o]=(s?"#"+s:":scope")+" "+xe(l[o]);c=l.join(",")}try{return H.apply(n,f.querySelectorAll(c)),n}catch(e){N(t,!0)}finally{s===S&&e.removeAttribute("id")}}}return g(t.replace($,"$1"),e,n,r)}function ue(){var r=[];return function e(t,n){return r.push(t+" ")>b.cacheLength&&delete e[r.shift()],e[t+" "]=n}}function le(e){return e[S]=!0,e}function ce(e){var t=C.createElement("fieldset");try{return!!e(t)}catch(e){return!1}finally{t.parentNode&&t.parentNode.removeChild(t),t=null}}function fe(e,t){var n=e.split("|"),r=n.length;while(r--)b.attrHandle[n[r]]=t}function pe(e,t){var n=t&&e,r=n&&1===e.nodeType&&1===t.nodeType&&e.sourceIndex-t.sourceIndex;if(r)return r;if(n)while(n=n.nextSibling)if(n===t)return-1;return e?1:-1}function de(t){return function(e){return"input"===e.nodeName.toLowerCase()&&e.type===t}}function he(n){return function(e){var t=e.nodeName.toLowerCase();return("input"===t||"button"===t)&&e.type===n}}function ge(t){return function(e){return"form"in e?e.parentNode&&!1===e.disabled?"label"in e?"label"in e.parentNode?e.parentNode.disabled===t:e.disabled===t:e.isDisabled===t||e.isDisabled!==!t&&ae(e)===t:e.disabled===t:"label"in e&&e.disabled===t}}function ve(a){return le(function(o){return o=+o,le(function(e,t){var n,r=a([],e.length,o),i=r.length;while(i--)e[n=r[i]]&&(e[n]=!(t[n]=e[n]))})})}function ye(e){return e&&"undefined"!=typeof e.getElementsByTagName&&e}for(e in d=se.support={},i=se.isXML=function(e){var t=e&&e.namespaceURI,n=e&&(e.ownerDocument||e).documentElement;return!Y.test(t||n&&n.nodeName||"HTML")},T=se.setDocument=function(e){var t,n,r=e?e.ownerDocument||e:p;return r!=C&&9===r.nodeType&&r.documentElement&&(a=(C=r).documentElement,E=!i(C),p!=C&&(n=C.defaultView)&&n.top!==n&&(n.addEventListener?n.addEventListener("unload",oe,!1):n.attachEvent&&n.attachEvent("onunload",oe)),d.scope=ce(function(e){return a.appendChild(e).appendChild(C.createElement("div")),"undefined"!=typeof e.querySelectorAll&&!e.querySelectorAll(":scope fieldset div").length}),d.attributes=ce(function(e){return e.className="i",!e.getAttribute("className")}),d.getElementsByTagName=ce(function(e){return e.appendChild(C.createComment("")),!e.getElementsByTagName("*").length}),d.getElementsByClassName=K.test(C.getElementsByClassName),d.getById=ce(function(e){return a.appendChild(e).id=S,!C.getElementsByName||!C.getElementsByName(S).length}),d.getById?(b.filter.ID=function(e){var t=e.replace(te,ne);return function(e){return e.getAttribute("id")===t}},b.find.ID=function(e,t){if("undefined"!=typeof t.getElementById&&E){var n=t.getElementById(e);return n?[n]:[]}}):(b.filter.ID=function(e){var n=e.replace(te,ne);return function(e){var t="undefined"!=typeof e.getAttributeNode&&e.getAttributeNode("id");return t&&t.value===n}},b.find.ID=function(e,t){if("undefined"!=typeof t.getElementById&&E){var n,r,i,o=t.getElementById(e);if(o){if((n=o.getAttributeNode("id"))&&n.value===e)return[o];i=t.getElementsByName(e),r=0;while(o=i[r++])if((n=o.getAttributeNode("id"))&&n.value===e)return[o]}return[]}}),b.find.TAG=d.getElementsByTagName?function(e,t){return"undefined"!=typeof t.getElementsByTagName?t.getElementsByTagName(e):d.qsa?t.querySelectorAll(e):void 0}:function(e,t){var n,r=[],i=0,o=t.getElementsByTagName(e);if("*"===e){while(n=o[i++])1===n.nodeType&&r.push(n);return r}return o},b.find.CLASS=d.getElementsByClassName&&function(e,t){if("undefined"!=typeof t.getElementsByClassName&&E)return t.getElementsByClassName(e)},s=[],v=[],(d.qsa=K.test(C.querySelectorAll))&&(ce(function(e){var t;a.appendChild(e).innerHTML="<a id='"+S+"'></a><select id='"+S+"-\r\\' msallowcapture=''><option selected=''></option></select>",e.querySelectorAll("[msallowcapture^='']").length&&v.push("[*^$]="+M+"*(?:''|\"\")"),e.querySelectorAll("[selected]").length||v.push("\\["+M+"*(?:value|"+R+")"),e.querySelectorAll("[id~="+S+"-]").length||v.push("~="),(t=C.createElement("input")).setAttribute("name",""),e.appendChild(t),e.querySelectorAll("[name='']").length||v.push("\\["+M+"*name"+M+"*="+M+"*(?:''|\"\")"),e.querySelectorAll(":checked").length||v.push(":checked"),e.querySelectorAll("a#"+S+"+*").length||v.push(".#.+[+~]"),e.querySelectorAll("\\\f"),v.push("[\\r\\n\\f]")}),ce(function(e){e.innerHTML="<a href='' disabled='disabled'></a><select disabled='disabled'><option/></select>";var t=C.createElement("input");t.setAttribute("type","hidden"),e.appendChild(t).setAttribute("name","D"),e.querySelectorAll("[name=d]").length&&v.push("name"+M+"*[*^$|!~]?="),2!==e.querySelectorAll(":enabled").length&&v.push(":enabled",":disabled"),a.appendChild(e).disabled=!0,2!==e.querySelectorAll(":disabled").length&&v.push(":enabled",":disabled"),e.querySelectorAll("*,:x"),v.push(",.*:")})),(d.matchesSelector=K.test(c=a.matches||a.webkitMatchesSelector||a.mozMatchesSelector||a.oMatchesSelector||a.msMatchesSelector))&&ce(function(e){d.disconnectedMatch=c.call(e,"*"),c.call(e,"[s!='']:x"),s.push("!=",F)}),v=v.length&&new RegExp(v.join("|")),s=s.length&&new RegExp(s.join("|")),t=K.test(a.compareDocumentPosition),y=t||K.test(a.contains)?function(e,t){var n=9===e.nodeType?e.documentElement:e,r=t&&t.parentNode;return e===r||!(!r||1!==r.nodeType||!(n.contains?n.contains(r):e.compareDocumentPosition&&16&e.compareDocumentPosition(r)))}:function(e,t){if(t)while(t=t.parentNode)if(t===e)return!0;return!1},j=t?function(e,t){if(e===t)return l=!0,0;var n=!e.compareDocumentPosition-!t.compareDocumentPosition;return n||(1&(n=(e.ownerDocument||e)==(t.ownerDocument||t)?e.compareDocumentPosition(t):1)||!d.sortDetached&&t.compareDocumentPosition(e)===n?e==C||e.ownerDocument==p&&y(p,e)?-1:t==C||t.ownerDocument==p&&y(p,t)?1:u?P(u,e)-P(u,t):0:4&n?-1:1)}:function(e,t){if(e===t)return l=!0,0;var n,r=0,i=e.parentNode,o=t.parentNode,a=[e],s=[t];if(!i||!o)return e==C?-1:t==C?1:i?-1:o?1:u?P(u,e)-P(u,t):0;if(i===o)return pe(e,t);n=e;while(n=n.parentNode)a.unshift(n);n=t;while(n=n.parentNode)s.unshift(n);while(a[r]===s[r])r++;return r?pe(a[r],s[r]):a[r]==p?-1:s[r]==p?1:0}),C},se.matches=function(e,t){return se(e,null,null,t)},se.matchesSelector=function(e,t){if(T(e),d.matchesSelector&&E&&!N[t+" "]&&(!s||!s.test(t))&&(!v||!v.test(t)))try{var n=c.call(e,t);if(n||d.disconnectedMatch||e.document&&11!==e.document.nodeType)return n}catch(e){N(t,!0)}return 0<se(t,C,null,[e]).length},se.contains=function(e,t){return(e.ownerDocument||e)!=C&&T(e),y(e,t)},se.attr=function(e,t){(e.ownerDocument||e)!=C&&T(e);var n=b.attrHandle[t.toLowerCase()],r=n&&D.call(b.attrHandle,t.toLowerCase())?n(e,t,!E):void 0;return void 0!==r?r:d.attributes||!E?e.getAttribute(t):(r=e.getAttributeNode(t))&&r.specified?r.value:null},se.escape=function(e){return(e+"").replace(re,ie)},se.error=function(e){throw new Error("Syntax error, unrecognized expression: "+e)},se.uniqueSort=function(e){var t,n=[],r=0,i=0;if(l=!d.detectDuplicates,u=!d.sortStable&&e.slice(0),e.sort(j),l){while(t=e[i++])t===e[i]&&(r=n.push(i));while(r--)e.splice(n[r],1)}return u=null,e},o=se.getText=function(e){var t,n="",r=0,i=e.nodeType;if(i){if(1===i||9===i||11===i){if("string"==typeof e.textContent)return e.textContent;for(e=e.firstChild;e;e=e.nextSibling)n+=o(e)}else if(3===i||4===i)return e.nodeValue}else while(t=e[r++])n+=o(t);return n},(b=se.selectors={cacheLength:50,createPseudo:le,match:G,attrHandle:{},find:{},relative:{">":{dir:"parentNode",first:!0}," ":{dir:"parentNode"},"+":{dir:"previousSibling",first:!0},"~":{dir:"previousSibling"}},preFilter:{ATTR:function(e){return e[1]=e[1].replace(te,ne),e[3]=(e[3]||e[4]||e[5]||"").replace(te,ne),"~="===e[2]&&(e[3]=" "+e[3]+" "),e.slice(0,4)},CHILD:function(e){return e[1]=e[1].toLowerCase(),"nth"===e[1].slice(0,3)?(e[3]||se.error(e[0]),e[4]=+(e[4]?e[5]+(e[6]||1):2*("even"===e[3]||"odd"===e[3])),e[5]=+(e[7]+e[8]||"odd"===e[3])):e[3]&&se.error(e[0]),e},PSEUDO:function(e){var t,n=!e[6]&&e[2];return G.CHILD.test(e[0])?null:(e[3]?e[2]=e[4]||e[5]||"":n&&X.test(n)&&(t=h(n,!0))&&(t=n.indexOf(")",n.length-t)-n.length)&&(e[0]=e[0].slice(0,t),e[2]=n.slice(0,t)),e.slice(0,3))}},filter:{TAG:function(e){var t=e.replace(te,ne).toLowerCase();return"*"===e?function(){return!0}:function(e){return e.nodeName&&e.nodeName.toLowerCase()===t}},CLASS:function(e){var t=m[e+" "];return t||(t=new RegExp("(^|"+M+")"+e+"("+M+"|$)"))&&m(e,function(e){return t.test("string"==typeof e.className&&e.className||"undefined"!=typeof e.getAttribute&&e.getAttribute("class")||"")})},ATTR:function(n,r,i){return function(e){var t=se.attr(e,n);return null==t?"!="===r:!r||(t+="","="===r?t===i:"!="===r?t!==i:"^="===r?i&&0===t.indexOf(i):"*="===r?i&&-1<t.indexOf(i):"$="===r?i&&t.slice(-i.length)===i:"~="===r?-1<(" "+t.replace(B," ")+" ").indexOf(i):"|="===r&&(t===i||t.slice(0,i.length+1)===i+"-"))}},CHILD:function(h,e,t,g,v){var y="nth"!==h.slice(0,3),m="last"!==h.slice(-4),x="of-type"===e;return 1===g&&0===v?function(e){return!!e.parentNode}:function(e,t,n){var r,i,o,a,s,u,l=y!==m?"nextSibling":"previousSibling",c=e.parentNode,f=x&&e.nodeName.toLowerCase(),p=!n&&!x,d=!1;if(c){if(y){while(l){a=e;while(a=a[l])if(x?a.nodeName.toLowerCase()===f:1===a.nodeType)return!1;u=l="only"===h&&!u&&"nextSibling"}return!0}if(u=[m?c.firstChild:c.lastChild],m&&p){d=(s=(r=(i=(o=(a=c)[S]||(a[S]={}))[a.uniqueID]||(o[a.uniqueID]={}))[h]||[])[0]===k&&r[1])&&r[2],a=s&&c.childNodes[s];while(a=++s&&a&&a[l]||(d=s=0)||u.pop())if(1===a.nodeType&&++d&&a===e){i[h]=[k,s,d];break}}else if(p&&(d=s=(r=(i=(o=(a=e)[S]||(a[S]={}))[a.uniqueID]||(o[a.uniqueID]={}))[h]||[])[0]===k&&r[1]),!1===d)while(a=++s&&a&&a[l]||(d=s=0)||u.pop())if((x?a.nodeName.toLowerCase()===f:1===a.nodeType)&&++d&&(p&&((i=(o=a[S]||(a[S]={}))[a.uniqueID]||(o[a.uniqueID]={}))[h]=[k,d]),a===e))break;return(d-=v)===g||d%g==0&&0<=d/g}}},PSEUDO:function(e,o){var t,a=b.pseudos[e]||b.setFilters[e.toLowerCase()]||se.error("unsupported pseudo: "+e);return a[S]?a(o):1<a.length?(t=[e,e,"",o],b.setFilters.hasOwnProperty(e.toLowerCase())?le(function(e,t){var n,r=a(e,o),i=r.length;while(i--)e[n=P(e,r[i])]=!(t[n]=r[i])}):function(e){return a(e,0,t)}):a}},pseudos:{not:le(function(e){var r=[],i=[],s=f(e.replace($,"$1"));return s[S]?le(function(e,t,n,r){var i,o=s(e,null,r,[]),a=e.length;while(a--)(i=o[a])&&(e[a]=!(t[a]=i))}):function(e,t,n){return r[0]=e,s(r,null,n,i),r[0]=null,!i.pop()}}),has:le(function(t){return function(e){return 0<se(t,e).length}}),contains:le(function(t){return t=t.replace(te,ne),function(e){return-1<(e.textContent||o(e)).indexOf(t)}}),lang:le(function(n){return V.test(n||"")||se.error("unsupported lang: "+n),n=n.replace(te,ne).toLowerCase(),function(e){var t;do{if(t=E?e.lang:e.getAttribute("xml:lang")||e.getAttribute("lang"))return(t=t.toLowerCase())===n||0===t.indexOf(n+"-")}while((e=e.parentNode)&&1===e.nodeType);return!1}}),target:function(e){var t=n.location&&n.location.hash;return t&&t.slice(1)===e.id},root:function(e){return e===a},focus:function(e){return e===C.activeElement&&(!C.hasFocus||C.hasFocus())&&!!(e.type||e.href||~e.tabIndex)},enabled:ge(!1),disabled:ge(!0),checked:function(e){var t=e.nodeName.toLowerCase();return"input"===t&&!!e.checked||"option"===t&&!!e.selected},selected:function(e){return e.parentNode&&e.parentNode.selectedIndex,!0===e.selected},empty:function(e){for(e=e.firstChild;e;e=e.nextSibling)if(e.nodeType<6)return!1;return!0},parent:function(e){return!b.pseudos.empty(e)},header:function(e){return J.test(e.nodeName)},input:function(e){return Q.test(e.nodeName)},button:function(e){var t=e.nodeName.toLowerCase();return"input"===t&&"button"===e.type||"button"===t},text:function(e){var t;return"input"===e.nodeName.toLowerCase()&&"text"===e.type&&(null==(t=e.getAttribute("type"))||"text"===t.toLowerCase())},first:ve(function(){return[0]}),last:ve(function(e,t){return[t-1]}),eq:ve(function(e,t,n){return[n<0?n+t:n]}),even:ve(function(e,t){for(var n=0;n<t;n+=2)e.push(n);return e}),odd:ve(function(e,t){for(var n=1;n<t;n+=2)e.push(n);return e}),lt:ve(function(e,t,n){for(var r=n<0?n+t:t<n?t:n;0<=--r;)e.push(r);return e}),gt:ve(function(e,t,n){for(var r=n<0?n+t:n;++r<t;)e.push(r);return e})}}).pseudos.nth=b.pseudos.eq,{radio:!0,checkbox:!0,file:!0,password:!0,image:!0})b.pseudos[e]=de(e);for(e in{submit:!0,reset:!0})b.pseudos[e]=he(e);function me(){}function xe(e){for(var t=0,n=e.length,r="";t<n;t++)r+=e[t].value;return r}function be(s,e,t){var u=e.dir,l=e.next,c=l||u,f=t&&"parentNode"===c,p=r++;return e.first?function(e,t,n){while(e=e[u])if(1===e.nodeType||f)return s(e,t,n);return!1}:function(e,t,n){var r,i,o,a=[k,p];if(n){while(e=e[u])if((1===e.nodeType||f)&&s(e,t,n))return!0}else while(e=e[u])if(1===e.nodeType||f)if(i=(o=e[S]||(e[S]={}))[e.uniqueID]||(o[e.uniqueID]={}),l&&l===e.nodeName.toLowerCase())e=e[u]||e;else{if((r=i[c])&&r[0]===k&&r[1]===p)return a[2]=r[2];if((i[c]=a)[2]=s(e,t,n))return!0}return!1}}function we(i){return 1<i.length?function(e,t,n){var r=i.length;while(r--)if(!i[r](e,t,n))return!1;return!0}:i[0]}function Te(e,t,n,r,i){for(var o,a=[],s=0,u=e.length,l=null!=t;s<u;s++)(o=e[s])&&(n&&!n(o,r,i)||(a.push(o),l&&t.push(s)));return a}function Ce(d,h,g,v,y,e){return v&&!v[S]&&(v=Ce(v)),y&&!y[S]&&(y=Ce(y,e)),le(function(e,t,n,r){var i,o,a,s=[],u=[],l=t.length,c=e||function(e,t,n){for(var r=0,i=t.length;r<i;r++)se(e,t[r],n);return n}(h||"*",n.nodeType?[n]:n,[]),f=!d||!e&&h?c:Te(c,s,d,n,r),p=g?y||(e?d:l||v)?[]:t:f;if(g&&g(f,p,n,r),v){i=Te(p,u),v(i,[],n,r),o=i.length;while(o--)(a=i[o])&&(p[u[o]]=!(f[u[o]]=a))}if(e){if(y||d){if(y){i=[],o=p.length;while(o--)(a=p[o])&&i.push(f[o]=a);y(null,p=[],i,r)}o=p.length;while(o--)(a=p[o])&&-1<(i=y?P(e,a):s[o])&&(e[i]=!(t[i]=a))}}else p=Te(p===t?p.splice(l,p.length):p),y?y(null,t,p,r):H.apply(t,p)})}function Ee(e){for(var i,t,n,r=e.length,o=b.relative[e[0].type],a=o||b.relative[" "],s=o?1:0,u=be(function(e){return e===i},a,!0),l=be(function(e){return-1<P(i,e)},a,!0),c=[function(e,t,n){var r=!o&&(n||t!==w)||((i=t).nodeType?u(e,t,n):l(e,t,n));return i=null,r}];s<r;s++)if(t=b.relative[e[s].type])c=[be(we(c),t)];else{if((t=b.filter[e[s].type].apply(null,e[s].matches))[S]){for(n=++s;n<r;n++)if(b.relative[e[n].type])break;return Ce(1<s&&we(c),1<s&&xe(e.slice(0,s-1).concat({value:" "===e[s-2].type?"*":""})).replace($,"$1"),t,s<n&&Ee(e.slice(s,n)),n<r&&Ee(e=e.slice(n)),n<r&&xe(e))}c.push(t)}return we(c)}return me.prototype=b.filters=b.pseudos,b.setFilters=new me,h=se.tokenize=function(e,t){var n,r,i,o,a,s,u,l=x[e+" "];if(l)return t?0:l.slice(0);a=e,s=[],u=b.preFilter;while(a){for(o in n&&!(r=_.exec(a))||(r&&(a=a.slice(r[0].length)||a),s.push(i=[])),n=!1,(r=z.exec(a))&&(n=r.shift(),i.push({value:n,type:r[0].replace($," ")}),a=a.slice(n.length)),b.filter)!(r=G[o].exec(a))||u[o]&&!(r=u[o](r))||(n=r.shift(),i.push({value:n,type:o,matches:r}),a=a.slice(n.length));if(!n)break}return t?a.length:a?se.error(e):x(e,s).slice(0)},f=se.compile=function(e,t){var n,v,y,m,x,r,i=[],o=[],a=A[e+" "];if(!a){t||(t=h(e)),n=t.length;while(n--)(a=Ee(t[n]))[S]?i.push(a):o.push(a);(a=A(e,(v=o,m=0<(y=i).length,x=0<v.length,r=function(e,t,n,r,i){var o,a,s,u=0,l="0",c=e&&[],f=[],p=w,d=e||x&&b.find.TAG("*",i),h=k+=null==p?1:Math.random()||.1,g=d.length;for(i&&(w=t==C||t||i);l!==g&&null!=(o=d[l]);l++){if(x&&o){a=0,t||o.ownerDocument==C||(T(o),n=!E);while(s=v[a++])if(s(o,t||C,n)){r.push(o);break}i&&(k=h)}m&&((o=!s&&o)&&u--,e&&c.push(o))}if(u+=l,m&&l!==u){a=0;while(s=y[a++])s(c,f,t,n);if(e){if(0<u)while(l--)c[l]||f[l]||(f[l]=q.call(r));f=Te(f)}H.apply(r,f),i&&!e&&0<f.length&&1<u+y.length&&se.uniqueSort(r)}return i&&(k=h,w=p),c},m?le(r):r))).selector=e}return a},g=se.select=function(e,t,n,r){var i,o,a,s,u,l="function"==typeof e&&e,c=!r&&h(e=l.selector||e);if(n=n||[],1===c.length){if(2<(o=c[0]=c[0].slice(0)).length&&"ID"===(a=o[0]).type&&9===t.nodeType&&E&&b.relative[o[1].type]){if(!(t=(b.find.ID(a.matches[0].replace(te,ne),t)||[])[0]))return n;l&&(t=t.parentNode),e=e.slice(o.shift().value.length)}i=G.needsContext.test(e)?0:o.length;while(i--){if(a=o[i],b.relative[s=a.type])break;if((u=b.find[s])&&(r=u(a.matches[0].replace(te,ne),ee.test(o[0].type)&&ye(t.parentNode)||t))){if(o.splice(i,1),!(e=r.length&&xe(o)))return H.apply(n,r),n;break}}}return(l||f(e,c))(r,t,!E,n,!t||ee.test(e)&&ye(t.parentNode)||t),n},d.sortStable=S.split("").sort(j).join("")===S,d.detectDuplicates=!!l,T(),d.sortDetached=ce(function(e){return 1&e.compareDocumentPosition(C.createElement("fieldset"))}),ce(function(e){return e.innerHTML="<a href='#'></a>","#"===e.firstChild.getAttribute("href")})||fe("type|href|height|width",function(e,t,n){if(!n)return e.getAttribute(t,"type"===t.toLowerCase()?1:2)}),d.attributes&&ce(function(e){return e.innerHTML="<input/>",e.firstChild.setAttribute("value",""),""===e.firstChild.getAttribute("value")})||fe("value",function(e,t,n){if(!n&&"input"===e.nodeName.toLowerCase())return e.defaultValue}),ce(function(e){return null==e.getAttribute("disabled")})||fe(R,function(e,t,n){var r;if(!n)return!0===e[t]?t.toLowerCase():(r=e.getAttributeNode(t))&&r.specified?r.value:null}),se}(C);S.find=d,S.expr=d.selectors,S.expr[":"]=S.expr.pseudos,S.uniqueSort=S.unique=d.uniqueSort,S.text=d.getText,S.isXMLDoc=d.isXML,S.contains=d.contains,S.escapeSelector=d.escape;var h=function(e,t,n){var r=[],i=void 0!==n;while((e=e[t])&&9!==e.nodeType)if(1===e.nodeType){if(i&&S(e).is(n))break;r.push(e)}return r},T=function(e,t){for(var n=[];e;e=e.nextSibling)1===e.nodeType&&e!==t&&n.push(e);return n},k=S.expr.match.needsContext;function A(e,t){return e.nodeName&&e.nodeName.toLowerCase()===t.toLowerCase()}var N=/^<([a-z][^\/\0>:\x20\t\r\n\f]*)[\x20\t\r\n\f]*\/?>(?:<\/\1>|)$/i;function j(e,n,r){return m(n)?S.grep(e,function(e,t){return!!n.call(e,t,e)!==r}):n.nodeType?S.grep(e,function(e){return e===n!==r}):"string"!=typeof n?S.grep(e,function(e){return-1<i.call(n,e)!==r}):S.filter(n,e,r)}S.filter=function(e,t,n){var r=t[0];return n&&(e=":not("+e+")"),1===t.length&&1===r.nodeType?S.find.matchesSelector(r,e)?[r]:[]:S.find.matches(e,S.grep(t,function(e){return 1===e.nodeType}))},S.fn.extend({find:function(e){var t,n,r=this.length,i=this;if("string"!=typeof e)return this.pushStack(S(e).filter(function(){for(t=0;t<r;t++)if(S.contains(i[t],this))return!0}));for(n=this.pushStack([]),t=0;t<r;t++)S.find(e,i[t],n);return 1<r?S.uniqueSort(n):n},filter:function(e){return this.pushStack(j(this,e||[],!1))},not:function(e){return this.pushStack(j(this,e||[],!0))},is:function(e){return!!j(this,"string"==typeof e&&k.test(e)?S(e):e||[],!1).length}});var D,q=/^(?:\s*(<[\w\W]+>)[^>]*|#([\w-]+))$/;(S.fn.init=function(e,t,n){var r,i;if(!e)return this;if(n=n||D,"string"==typeof e){if(!(r="<"===e[0]&&">"===e[e.length-1]&&3<=e.length?[null,e,null]:q.exec(e))||!r[1]&&t)return!t||t.jquery?(t||n).find(e):this.constructor(t).find(e);if(r[1]){if(t=t instanceof S?t[0]:t,S.merge(this,S.parseHTML(r[1],t&&t.nodeType?t.ownerDocument||t:E,!0)),N.test(r[1])&&S.isPlainObject(t))for(r in t)m(this[r])?this[r](t[r]):this.attr(r,t[r]);return this}return(i=E.getElementById(r[2]))&&(this[0]=i,this.length=1),this}return e.nodeType?(this[0]=e,this.length=1,this):m(e)?void 0!==n.ready?n.ready(e):e(S):S.makeArray(e,this)}).prototype=S.fn,D=S(E);var L=/^(?:parents|prev(?:Until|All))/,H={children:!0,contents:!0,next:!0,prev:!0};function O(e,t){while((e=e[t])&&1!==e.nodeType);return e}S.fn.extend({has:function(e){var t=S(e,this),n=t.length;return this.filter(function(){for(var e=0;e<n;e++)if(S.contains(this,t[e]))return!0})},closest:function(e,t){var n,r=0,i=this.length,o=[],a="string"!=typeof e&&S(e);if(!k.test(e))for(;r<i;r++)for(n=this[r];n&&n!==t;n=n.parentNode)if(n.nodeType<11&&(a?-1<a.index(n):1===n.nodeType&&S.find.matchesSelector(n,e))){o.push(n);break}return this.pushStack(1<o.length?S.uniqueSort(o):o)},index:function(e){return e?"string"==typeof e?i.call(S(e),this[0]):i.call(this,e.jquery?e[0]:e):this[0]&&this[0].parentNode?this.first().prevAll().length:-1},add:function(e,t){return this.pushStack(S.uniqueSort(S.merge(this.get(),S(e,t))))},addBack:function(e){return this.add(null==e?this.prevObject:this.prevObject.filter(e))}}),S.each({parent:function(e){var t=e.parentNode;return t&&11!==t.nodeType?t:null},parents:function(e){return h(e,"parentNode")},parentsUntil:function(e,t,n){return h(e,"parentNode",n)},next:function(e){return O(e,"nextSibling")},prev:function(e){return O(e,"previousSibling")},nextAll:function(e){return h(e,"nextSibling")},prevAll:function(e){return h(e,"previousSibling")},nextUntil:function(e,t,n){return h(e,"nextSibling",n)},prevUntil:function(e,t,n){return h(e,"previousSibling",n)},siblings:function(e){return T((e.parentNode||{}).firstChild,e)},children:function(e){return T(e.firstChild)},contents:function(e){return null!=e.contentDocument&&r(e.contentDocument)?e.contentDocument:(A(e,"template")&&(e=e.content||e),S.merge([],e.childNodes))}},function(r,i){S.fn[r]=function(e,t){var n=S.map(this,i,e);return"Until"!==r.slice(-5)&&(t=e),t&&"string"==typeof t&&(n=S.filter(t,n)),1<this.length&&(H[r]||S.uniqueSort(n),L.test(r)&&n.reverse()),this.pushStack(n)}});var P=/[^\x20\t\r\n\f]+/g;function R(e){return e}function M(e){throw e}function I(e,t,n,r){var i;try{e&&m(i=e.promise)?i.call(e).done(t).fail(n):e&&m(i=e.then)?i.call(e,t,n):t.apply(void 0,[e].slice(r))}catch(e){n.apply(void 0,[e])}}S.Callbacks=function(r){var e,n;r="string"==typeof r?(e=r,n={},S.each(e.match(P)||[],function(e,t){n[t]=!0}),n):S.extend({},r);var i,t,o,a,s=[],u=[],l=-1,c=function(){for(a=a||r.once,o=i=!0;u.length;l=-1){t=u.shift();while(++l<s.length)!1===s[l].apply(t[0],t[1])&&r.stopOnFalse&&(l=s.length,t=!1)}r.memory||(t=!1),i=!1,a&&(s=t?[]:"")},f={add:function(){return s&&(t&&!i&&(l=s.length-1,u.push(t)),function n(e){S.each(e,function(e,t){m(t)?r.unique&&f.has(t)||s.push(t):t&&t.length&&"string"!==w(t)&&n(t)})}(arguments),t&&!i&&c()),this},remove:function(){return S.each(arguments,function(e,t){var n;while(-1<(n=S.inArray(t,s,n)))s.splice(n,1),n<=l&&l--}),this},has:function(e){return e?-1<S.inArray(e,s):0<s.length},empty:function(){return s&&(s=[]),this},disable:function(){return a=u=[],s=t="",this},disabled:function(){return!s},lock:function(){return a=u=[],t||i||(s=t=""),this},locked:function(){return!!a},fireWith:function(e,t){return a||(t=[e,(t=t||[]).slice?t.slice():t],u.push(t),i||c()),this},fire:function(){return f.fireWith(this,arguments),this},fired:function(){return!!o}};return f},S.extend({Deferred:function(e){var o=[["notify","progress",S.Callbacks("memory"),S.Callbacks("memory"),2],["resolve","done",S.Callbacks("once memory"),S.Callbacks("once memory"),0,"resolved"],["reject","fail",S.Callbacks("once memory"),S.Callbacks("once memory"),1,"rejected"]],i="pending",a={state:function(){return i},always:function(){return s.done(arguments).fail(arguments),this},"catch":function(e){return a.then(null,e)},pipe:function(){var i=arguments;return S.Deferred(function(r){S.each(o,function(e,t){var n=m(i[t[4]])&&i[t[4]];s[t[1]](function(){var e=n&&n.apply(this,arguments);e&&m(e.promise)?e.promise().progress(r.notify).done(r.resolve).fail(r.reject):r[t[0]+"With"](this,n?[e]:arguments)})}),i=null}).promise()},then:function(t,n,r){var u=0;function l(i,o,a,s){return function(){var n=this,r=arguments,e=function(){var e,t;if(!(i<u)){if((e=a.apply(n,r))===o.promise())throw new TypeError("Thenable self-resolution");t=e&&("object"==typeof e||"function"==typeof e)&&e.then,m(t)?s?t.call(e,l(u,o,R,s),l(u,o,M,s)):(u++,t.call(e,l(u,o,R,s),l(u,o,M,s),l(u,o,R,o.notifyWith))):(a!==R&&(n=void 0,r=[e]),(s||o.resolveWith)(n,r))}},t=s?e:function(){try{e()}catch(e){S.Deferred.exceptionHook&&S.Deferred.exceptionHook(e,t.stackTrace),u<=i+1&&(a!==M&&(n=void 0,r=[e]),o.rejectWith(n,r))}};i?t():(S.Deferred.getStackHook&&(t.stackTrace=S.Deferred.getStackHook()),C.setTimeout(t))}}return S.Deferred(function(e){o[0][3].add(l(0,e,m(r)?r:R,e.notifyWith)),o[1][3].add(l(0,e,m(t)?t:R)),o[2][3].add(l(0,e,m(n)?n:M))}).promise()},promise:function(e){return null!=e?S.extend(e,a):a}},s={};return S.each(o,function(e,t){var n=t[2],r=t[5];a[t[1]]=n.add,r&&n.add(function(){i=r},o[3-e][2].disable,o[3-e][3].disable,o[0][2].lock,o[0][3].lock),n.add(t[3].fire),s[t[0]]=function(){return s[t[0]+"With"](this===s?void 0:this,arguments),this},s[t[0]+"With"]=n.fireWith}),a.promise(s),e&&e.call(s,s),s},when:function(e){var n=arguments.length,t=n,r=Array(t),i=s.call(arguments),o=S.Deferred(),a=function(t){return function(e){r[t]=this,i[t]=1<arguments.length?s.call(arguments):e,--n||o.resolveWith(r,i)}};if(n<=1&&(I(e,o.done(a(t)).resolve,o.reject,!n),"pending"===o.state()||m(i[t]&&i[t].then)))return o.then();while(t--)I(i[t],a(t),o.reject);return o.promise()}});var W=/^(Eval|Internal|Range|Reference|Syntax|Type|URI)Error$/;S.Deferred.exceptionHook=function(e,t){C.console&&C.console.warn&&e&&W.test(e.name)&&C.console.warn("jQuery.Deferred exception: "+e.message,e.stack,t)},S.readyException=function(e){C.setTimeout(function(){throw e})};var F=S.Deferred();function B(){E.removeEventListener("DOMContentLoaded",B),C.removeEventListener("load",B),S.ready()}S.fn.ready=function(e){return F.then(e)["catch"](function(e){S.readyException(e)}),this},S.extend({isReady:!1,readyWait:1,ready:function(e){(!0===e?--S.readyWait:S.isReady)||(S.isReady=!0)!==e&&0<--S.readyWait||F.resolveWith(E,[S])}}),S.ready.then=F.then,"complete"===E.readyState||"loading"!==E.readyState&&!E.documentElement.doScroll?C.setTimeout(S.ready):(E.addEventListener("DOMContentLoaded",B),C.addEventListener("load",B));var $=function(e,t,n,r,i,o,a){var s=0,u=e.length,l=null==n;if("object"===w(n))for(s in i=!0,n)$(e,t,s,n[s],!0,o,a);else if(void 0!==r&&(i=!0,m(r)||(a=!0),l&&(a?(t.call(e,r),t=null):(l=t,t=function(e,t,n){return l.call(S(e),n)})),t))for(;s<u;s++)t(e[s],n,a?r:r.call(e[s],s,t(e[s],n)));return i?e:l?t.call(e):u?t(e[0],n):o},_=/^-ms-/,z=/-([a-z])/g;function U(e,t){return t.toUpperCase()}function X(e){return e.replace(_,"ms-").replace(z,U)}var V=function(e){return 1===e.nodeType||9===e.nodeType||!+e.nodeType};function G(){this.expando=S.expando+G.uid++}G.uid=1,G.prototype={cache:function(e){var t=e[this.expando];return t||(t={},V(e)&&(e.nodeType?e[this.expando]=t:Object.defineProperty(e,this.expando,{value:t,configurable:!0}))),t},set:function(e,t,n){var r,i=this.cache(e);if("string"==typeof t)i[X(t)]=n;else for(r in t)i[X(r)]=t[r];return i},get:function(e,t){return void 0===t?this.cache(e):e[this.expando]&&e[this.expando][X(t)]},access:function(e,t,n){return void 0===t||t&&"string"==typeof t&&void 0===n?this.get(e,t):(this.set(e,t,n),void 0!==n?n:t)},remove:function(e,t){var n,r=e[this.expando];if(void 0!==r){if(void 0!==t){n=(t=Array.isArray(t)?t.map(X):(t=X(t))in r?[t]:t.match(P)||[]).length;while(n--)delete r[t[n]]}(void 0===t||S.isEmptyObject(r))&&(e.nodeType?e[this.expando]=void 0:delete e[this.expando])}},hasData:function(e){var t=e[this.expando];return void 0!==t&&!S.isEmptyObject(t)}};var Y=new G,Q=new G,J=/^(?:\{[\w\W]*\}|\[[\w\W]*\])$/,K=/[A-Z]/g;function Z(e,t,n){var r,i;if(void 0===n&&1===e.nodeType)if(r="data-"+t.replace(K,"-$&").toLowerCase(),"string"==typeof(n=e.getAttribute(r))){try{n="true"===(i=n)||"false"!==i&&("null"===i?null:i===+i+""?+i:J.test(i)?JSON.parse(i):i)}catch(e){}Q.set(e,t,n)}else n=void 0;return n}S.extend({hasData:function(e){return Q.hasData(e)||Y.hasData(e)},data:function(e,t,n){return Q.access(e,t,n)},removeData:function(e,t){Q.remove(e,t)},_data:function(e,t,n){return Y.access(e,t,n)},_removeData:function(e,t){Y.remove(e,t)}}),S.fn.extend({data:function(n,e){var t,r,i,o=this[0],a=o&&o.attributes;if(void 0===n){if(this.length&&(i=Q.get(o),1===o.nodeType&&!Y.get(o,"hasDataAttrs"))){t=a.length;while(t--)a[t]&&0===(r=a[t].name).indexOf("data-")&&(r=X(r.slice(5)),Z(o,r,i[r]));Y.set(o,"hasDataAttrs",!0)}return i}return"object"==typeof n?this.each(function(){Q.set(this,n)}):$(this,function(e){var t;if(o&&void 0===e)return void 0!==(t=Q.get(o,n))?t:void 0!==(t=Z(o,n))?t:void 0;this.each(function(){Q.set(this,n,e)})},null,e,1<arguments.length,null,!0)},removeData:function(e){return this.each(function(){Q.remove(this,e)})}}),S.extend({queue:function(e,t,n){var r;if(e)return t=(t||"fx")+"queue",r=Y.get(e,t),n&&(!r||Array.isArray(n)?r=Y.access(e,t,S.makeArray(n)):r.push(n)),r||[]},dequeue:function(e,t){t=t||"fx";var n=S.queue(e,t),r=n.length,i=n.shift(),o=S._queueHooks(e,t);"inprogress"===i&&(i=n.shift(),r--),i&&("fx"===t&&n.unshift("inprogress"),delete o.stop,i.call(e,function(){S.dequeue(e,t)},o)),!r&&o&&o.empty.fire()},_queueHooks:function(e,t){var n=t+"queueHooks";return Y.get(e,n)||Y.access(e,n,{empty:S.Callbacks("once memory").add(function(){Y.remove(e,[t+"queue",n])})})}}),S.fn.extend({queue:function(t,n){var e=2;return"string"!=typeof t&&(n=t,t="fx",e--),arguments.length<e?S.queue(this[0],t):void 0===n?this:this.each(function(){var e=S.queue(this,t,n);S._queueHooks(this,t),"fx"===t&&"inprogress"!==e[0]&&S.dequeue(this,t)})},dequeue:function(e){return this.each(function(){S.dequeue(this,e)})},clearQueue:function(e){return this.queue(e||"fx",[])},promise:function(e,t){var n,r=1,i=S.Deferred(),o=this,a=this.length,s=function(){--r||i.resolveWith(o,[o])};"string"!=typeof e&&(t=e,e=void 0),e=e||"fx";while(a--)(n=Y.get(o[a],e+"queueHooks"))&&n.empty&&(r++,n.empty.add(s));return s(),i.promise(t)}});var ee=/[+-]?(?:\d*\.|)\d+(?:[eE][+-]?\d+|)/.source,te=new RegExp("^(?:([+-])=|)("+ee+")([a-z%]*)$","i"),ne=["Top","Right","Bottom","Left"],re=E.documentElement,ie=function(e){return S.contains(e.ownerDocument,e)},oe={composed:!0};re.getRootNode&&(ie=function(e){return S.contains(e.ownerDocument,e)||e.getRootNode(oe)===e.ownerDocument});var ae=function(e,t){return"none"===(e=t||e).style.display||""===e.style.display&&ie(e)&&"none"===S.css(e,"display")};function se(e,t,n,r){var i,o,a=20,s=r?function(){return r.cur()}:function(){return S.css(e,t,"")},u=s(),l=n&&n[3]||(S.cssNumber[t]?"":"px"),c=e.nodeType&&(S.cssNumber[t]||"px"!==l&&+u)&&te.exec(S.css(e,t));if(c&&c[3]!==l){u/=2,l=l||c[3],c=+u||1;while(a--)S.style(e,t,c+l),(1-o)*(1-(o=s()/u||.5))<=0&&(a=0),c/=o;c*=2,S.style(e,t,c+l),n=n||[]}return n&&(c=+c||+u||0,i=n[1]?c+(n[1]+1)*n[2]:+n[2],r&&(r.unit=l,r.start=c,r.end=i)),i}var ue={};function le(e,t){for(var n,r,i,o,a,s,u,l=[],c=0,f=e.length;c<f;c++)(r=e[c]).style&&(n=r.style.display,t?("none"===n&&(l[c]=Y.get(r,"display")||null,l[c]||(r.style.display="")),""===r.style.display&&ae(r)&&(l[c]=(u=a=o=void 0,a=(i=r).ownerDocument,s=i.nodeName,(u=ue[s])||(o=a.body.appendChild(a.createElement(s)),u=S.css(o,"display"),o.parentNode.removeChild(o),"none"===u&&(u="block"),ue[s]=u)))):"none"!==n&&(l[c]="none",Y.set(r,"display",n)));for(c=0;c<f;c++)null!=l[c]&&(e[c].style.display=l[c]);return e}S.fn.extend({show:function(){return le(this,!0)},hide:function(){return le(this)},toggle:function(e){return"boolean"==typeof e?e?this.show():this.hide():this.each(function(){ae(this)?S(this).show():S(this).hide()})}});var ce,fe,pe=/^(?:checkbox|radio)$/i,de=/<([a-z][^\/\0>\x20\t\r\n\f]*)/i,he=/^$|^module$|\/(?:java|ecma)script/i;ce=E.createDocumentFragment().appendChild(E.createElement("div")),(fe=E.createElement("input")).setAttribute("type","radio"),fe.setAttribute("checked","checked"),fe.setAttribute("name","t"),ce.appendChild(fe),y.checkClone=ce.cloneNode(!0).cloneNode(!0).lastChild.checked,ce.innerHTML="<textarea>x</textarea>",y.noCloneChecked=!!ce.cloneNode(!0).lastChild.defaultValue,ce.innerHTML="<option></option>",y.option=!!ce.lastChild;var ge={thead:[1,"<table>","</table>"],col:[2,"<table><colgroup>","</colgroup></table>"],tr:[2,"<table><tbody>","</tbody></table>"],td:[3,"<table><tbody><tr>","</tr></tbody></table>"],_default:[0,"",""]};function ve(e,t){var n;return n="undefined"!=typeof e.getElementsByTagName?e.getElementsByTagName(t||"*"):"undefined"!=typeof e.querySelectorAll?e.querySelectorAll(t||"*"):[],void 0===t||t&&A(e,t)?S.merge([e],n):n}function ye(e,t){for(var n=0,r=e.length;n<r;n++)Y.set(e[n],"globalEval",!t||Y.get(t[n],"globalEval"))}ge.tbody=ge.tfoot=ge.colgroup=ge.caption=ge.thead,ge.th=ge.td,y.option||(ge.optgroup=ge.option=[1,"<select multiple='multiple'>","</select>"]);var me=/<|&#?\w+;/;function xe(e,t,n,r,i){for(var o,a,s,u,l,c,f=t.createDocumentFragment(),p=[],d=0,h=e.length;d<h;d++)if((o=e[d])||0===o)if("object"===w(o))S.merge(p,o.nodeType?[o]:o);else if(me.test(o)){a=a||f.appendChild(t.createElement("div")),s=(de.exec(o)||["",""])[1].toLowerCase(),u=ge[s]||ge._default,a.innerHTML=u[1]+S.htmlPrefilter(o)+u[2],c=u[0];while(c--)a=a.lastChild;S.merge(p,a.childNodes),(a=f.firstChild).textContent=""}else p.push(t.createTextNode(o));f.textContent="",d=0;while(o=p[d++])if(r&&-1<S.inArray(o,r))i&&i.push(o);else if(l=ie(o),a=ve(f.appendChild(o),"script"),l&&ye(a),n){c=0;while(o=a[c++])he.test(o.type||"")&&n.push(o)}return f}var be=/^([^.]*)(?:\.(.+)|)/;function we(){return!0}function Te(){return!1}function Ce(e,t){return e===function(){try{return E.activeElement}catch(e){}}()==("focus"===t)}function Ee(e,t,n,r,i,o){var a,s;if("object"==typeof t){for(s in"string"!=typeof n&&(r=r||n,n=void 0),t)Ee(e,s,n,r,t[s],o);return e}if(null==r&&null==i?(i=n,r=n=void 0):null==i&&("string"==typeof n?(i=r,r=void 0):(i=r,r=n,n=void 0)),!1===i)i=Te;else if(!i)return e;return 1===o&&(a=i,(i=function(e){return S().off(e),a.apply(this,arguments)}).guid=a.guid||(a.guid=S.guid++)),e.each(function(){S.event.add(this,t,i,r,n)})}function Se(e,i,o){o?(Y.set(e,i,!1),S.event.add(e,i,{namespace:!1,handler:function(e){var t,n,r=Y.get(this,i);if(1&e.isTrigger&&this[i]){if(r.length)(S.event.special[i]||{}).delegateType&&e.stopPropagation();else if(r=s.call(arguments),Y.set(this,i,r),t=o(this,i),this[i](),r!==(n=Y.get(this,i))||t?Y.set(this,i,!1):n={},r!==n)return e.stopImmediatePropagation(),e.preventDefault(),n&&n.value}else r.length&&(Y.set(this,i,{value:S.event.trigger(S.extend(r[0],S.Event.prototype),r.slice(1),this)}),e.stopImmediatePropagation())}})):void 0===Y.get(e,i)&&S.event.add(e,i,we)}S.event={global:{},add:function(t,e,n,r,i){var o,a,s,u,l,c,f,p,d,h,g,v=Y.get(t);if(V(t)){n.handler&&(n=(o=n).handler,i=o.selector),i&&S.find.matchesSelector(re,i),n.guid||(n.guid=S.guid++),(u=v.events)||(u=v.events=Object.create(null)),(a=v.handle)||(a=v.handle=function(e){return"undefined"!=typeof S&&S.event.triggered!==e.type?S.event.dispatch.apply(t,arguments):void 0}),l=(e=(e||"").match(P)||[""]).length;while(l--)d=g=(s=be.exec(e[l])||[])[1],h=(s[2]||"").split(".").sort(),d&&(f=S.event.special[d]||{},d=(i?f.delegateType:f.bindType)||d,f=S.event.special[d]||{},c=S.extend({type:d,origType:g,data:r,handler:n,guid:n.guid,selector:i,needsContext:i&&S.expr.match.needsContext.test(i),namespace:h.join(".")},o),(p=u[d])||((p=u[d]=[]).delegateCount=0,f.setup&&!1!==f.setup.call(t,r,h,a)||t.addEventListener&&t.addEventListener(d,a)),f.add&&(f.add.call(t,c),c.handler.guid||(c.handler.guid=n.guid)),i?p.splice(p.delegateCount++,0,c):p.push(c),S.event.global[d]=!0)}},remove:function(e,t,n,r,i){var o,a,s,u,l,c,f,p,d,h,g,v=Y.hasData(e)&&Y.get(e);if(v&&(u=v.events)){l=(t=(t||"").match(P)||[""]).length;while(l--)if(d=g=(s=be.exec(t[l])||[])[1],h=(s[2]||"").split(".").sort(),d){f=S.event.special[d]||{},p=u[d=(r?f.delegateType:f.bindType)||d]||[],s=s[2]&&new RegExp("(^|\\.)"+h.join("\\.(?:.*\\.|)")+"(\\.|$)"),a=o=p.length;while(o--)c=p[o],!i&&g!==c.origType||n&&n.guid!==c.guid||s&&!s.test(c.namespace)||r&&r!==c.selector&&("**"!==r||!c.selector)||(p.splice(o,1),c.selector&&p.delegateCount--,f.remove&&f.remove.call(e,c));a&&!p.length&&(f.teardown&&!1!==f.teardown.call(e,h,v.handle)||S.removeEvent(e,d,v.handle),delete u[d])}else for(d in u)S.event.remove(e,d+t[l],n,r,!0);S.isEmptyObject(u)&&Y.remove(e,"handle events")}},dispatch:function(e){var t,n,r,i,o,a,s=new Array(arguments.length),u=S.event.fix(e),l=(Y.get(this,"events")||Object.create(null))[u.type]||[],c=S.event.special[u.type]||{};for(s[0]=u,t=1;t<arguments.length;t++)s[t]=arguments[t];if(u.delegateTarget=this,!c.preDispatch||!1!==c.preDispatch.call(this,u)){a=S.event.handlers.call(this,u,l),t=0;while((i=a[t++])&&!u.isPropagationStopped()){u.currentTarget=i.elem,n=0;while((o=i.handlers[n++])&&!u.isImmediatePropagationStopped())u.rnamespace&&!1!==o.namespace&&!u.rnamespace.test(o.namespace)||(u.handleObj=o,u.data=o.data,void 0!==(r=((S.event.special[o.origType]||{}).handle||o.handler).apply(i.elem,s))&&!1===(u.result=r)&&(u.preventDefault(),u.stopPropagation()))}return c.postDispatch&&c.postDispatch.call(this,u),u.result}},handlers:function(e,t){var n,r,i,o,a,s=[],u=t.delegateCount,l=e.target;if(u&&l.nodeType&&!("click"===e.type&&1<=e.button))for(;l!==this;l=l.parentNode||this)if(1===l.nodeType&&("click"!==e.type||!0!==l.disabled)){for(o=[],a={},n=0;n<u;n++)void 0===a[i=(r=t[n]).selector+" "]&&(a[i]=r.needsContext?-1<S(i,this).index(l):S.find(i,this,null,[l]).length),a[i]&&o.push(r);o.length&&s.push({elem:l,handlers:o})}return l=this,u<t.length&&s.push({elem:l,handlers:t.slice(u)}),s},addProp:function(t,e){Object.defineProperty(S.Event.prototype,t,{enumerable:!0,configurable:!0,get:m(e)?function(){if(this.originalEvent)return e(this.originalEvent)}:function(){if(this.originalEvent)return this.originalEvent[t]},set:function(e){Object.defineProperty(this,t,{enumerable:!0,configurable:!0,writable:!0,value:e})}})},fix:function(e){return e[S.expando]?e:new S.Event(e)},special:{load:{noBubble:!0},click:{setup:function(e){var t=this||e;return pe.test(t.type)&&t.click&&A(t,"input")&&Se(t,"click",we),!1},trigger:function(e){var t=this||e;return pe.test(t.type)&&t.click&&A(t,"input")&&Se(t,"click"),!0},_default:function(e){var t=e.target;return pe.test(t.type)&&t.click&&A(t,"input")&&Y.get(t,"click")||A(t,"a")}},beforeunload:{postDispatch:function(e){void 0!==e.result&&e.originalEvent&&(e.originalEvent.returnValue=e.result)}}}},S.removeEvent=function(e,t,n){e.removeEventListener&&e.removeEventListener(t,n)},S.Event=function(e,t){if(!(this instanceof S.Event))return new S.Event(e,t);e&&e.type?(this.originalEvent=e,this.type=e.type,this.isDefaultPrevented=e.defaultPrevented||void 0===e.defaultPrevented&&!1===e.returnValue?we:Te,this.target=e.target&&3===e.target.nodeType?e.target.parentNode:e.target,this.currentTarget=e.currentTarget,this.relatedTarget=e.relatedTarget):this.type=e,t&&S.extend(this,t),this.timeStamp=e&&e.timeStamp||Date.now(),this[S.expando]=!0},S.Event.prototype={constructor:S.Event,isDefaultPrevented:Te,isPropagationStopped:Te,isImmediatePropagationStopped:Te,isSimulated:!1,preventDefault:function(){var e=this.originalEvent;this.isDefaultPrevented=we,e&&!this.isSimulated&&e.preventDefault()},stopPropagation:function(){var e=this.originalEvent;this.isPropagationStopped=we,e&&!this.isSimulated&&e.stopPropagation()},stopImmediatePropagation:function(){var e=this.originalEvent;this.isImmediatePropagationStopped=we,e&&!this.isSimulated&&e.stopImmediatePropagation(),this.stopPropagation()}},S.each({altKey:!0,bubbles:!0,cancelable:!0,changedTouches:!0,ctrlKey:!0,detail:!0,eventPhase:!0,metaKey:!0,pageX:!0,pageY:!0,shiftKey:!0,view:!0,"char":!0,code:!0,charCode:!0,key:!0,keyCode:!0,button:!0,buttons:!0,clientX:!0,clientY:!0,offsetX:!0,offsetY:!0,pointerId:!0,pointerType:!0,screenX:!0,screenY:!0,targetTouches:!0,toElement:!0,touches:!0,which:!0},S.event.addProp),S.each({focus:"focusin",blur:"focusout"},function(e,t){S.event.special[e]={setup:function(){return Se(this,e,Ce),!1},trigger:function(){return Se(this,e),!0},_default:function(){return!0},delegateType:t}}),S.each({mouseenter:"mouseover",mouseleave:"mouseout",pointerenter:"pointerover",pointerleave:"pointerout"},function(e,i){S.event.special[e]={delegateType:i,bindType:i,handle:function(e){var t,n=e.relatedTarget,r=e.handleObj;return n&&(n===this||S.contains(this,n))||(e.type=r.origType,t=r.handler.apply(this,arguments),e.type=i),t}}}),S.fn.extend({on:function(e,t,n,r){return Ee(this,e,t,n,r)},one:function(e,t,n,r){return Ee(this,e,t,n,r,1)},off:function(e,t,n){var r,i;if(e&&e.preventDefault&&e.handleObj)return r=e.handleObj,S(e.delegateTarget).off(r.namespace?r.origType+"."+r.namespace:r.origType,r.selector,r.handler),this;if("object"==typeof e){for(i in e)this.off(i,t,e[i]);return this}return!1!==t&&"function"!=typeof t||(n=t,t=void 0),!1===n&&(n=Te),this.each(function(){S.event.remove(this,e,n,t)})}});var ke=/<script|<style|<link/i,Ae=/checked\s*(?:[^=]|=\s*.checked.)/i,Ne=/^\s*<!(?:\[CDATA\[|--)|(?:\]\]|--)>\s*$/g;function je(e,t){return A(e,"table")&&A(11!==t.nodeType?t:t.firstChild,"tr")&&S(e).children("tbody")[0]||e}function De(e){return e.type=(null!==e.getAttribute("type"))+"/"+e.type,e}function qe(e){return"true/"===(e.type||"").slice(0,5)?e.type=e.type.slice(5):e.removeAttribute("type"),e}function Le(e,t){var n,r,i,o,a,s;if(1===t.nodeType){if(Y.hasData(e)&&(s=Y.get(e).events))for(i in Y.remove(t,"handle events"),s)for(n=0,r=s[i].length;n<r;n++)S.event.add(t,i,s[i][n]);Q.hasData(e)&&(o=Q.access(e),a=S.extend({},o),Q.set(t,a))}}function He(n,r,i,o){r=g(r);var e,t,a,s,u,l,c=0,f=n.length,p=f-1,d=r[0],h=m(d);if(h||1<f&&"string"==typeof d&&!y.checkClone&&Ae.test(d))return n.each(function(e){var t=n.eq(e);h&&(r[0]=d.call(this,e,t.html())),He(t,r,i,o)});if(f&&(t=(e=xe(r,n[0].ownerDocument,!1,n,o)).firstChild,1===e.childNodes.length&&(e=t),t||o)){for(s=(a=S.map(ve(e,"script"),De)).length;c<f;c++)u=e,c!==p&&(u=S.clone(u,!0,!0),s&&S.merge(a,ve(u,"script"))),i.call(n[c],u,c);if(s)for(l=a[a.length-1].ownerDocument,S.map(a,qe),c=0;c<s;c++)u=a[c],he.test(u.type||"")&&!Y.access(u,"globalEval")&&S.contains(l,u)&&(u.src&&"module"!==(u.type||"").toLowerCase()?S._evalUrl&&!u.noModule&&S._evalUrl(u.src,{nonce:u.nonce||u.getAttribute("nonce")},l):b(u.textContent.replace(Ne,""),u,l))}return n}function Oe(e,t,n){for(var r,i=t?S.filter(t,e):e,o=0;null!=(r=i[o]);o++)n||1!==r.nodeType||S.cleanData(ve(r)),r.parentNode&&(n&&ie(r)&&ye(ve(r,"script")),r.parentNode.removeChild(r));return e}S.extend({htmlPrefilter:function(e){return e},clone:function(e,t,n){var r,i,o,a,s,u,l,c=e.cloneNode(!0),f=ie(e);if(!(y.noCloneChecked||1!==e.nodeType&&11!==e.nodeType||S.isXMLDoc(e)))for(a=ve(c),r=0,i=(o=ve(e)).length;r<i;r++)s=o[r],u=a[r],void 0,"input"===(l=u.nodeName.toLowerCase())&&pe.test(s.type)?u.checked=s.checked:"input"!==l&&"textarea"!==l||(u.defaultValue=s.defaultValue);if(t)if(n)for(o=o||ve(e),a=a||ve(c),r=0,i=o.length;r<i;r++)Le(o[r],a[r]);else Le(e,c);return 0<(a=ve(c,"script")).length&&ye(a,!f&&ve(e,"script")),c},cleanData:function(e){for(var t,n,r,i=S.event.special,o=0;void 0!==(n=e[o]);o++)if(V(n)){if(t=n[Y.expando]){if(t.events)for(r in t.events)i[r]?S.event.remove(n,r):S.removeEvent(n,r,t.handle);n[Y.expando]=void 0}n[Q.expando]&&(n[Q.expando]=void 0)}}}),S.fn.extend({detach:function(e){return Oe(this,e,!0)},remove:function(e){return Oe(this,e)},text:function(e){return $(this,function(e){return void 0===e?S.text(this):this.empty().each(function(){1!==this.nodeType&&11!==this.nodeType&&9!==this.nodeType||(this.textContent=e)})},null,e,arguments.length)},append:function(){return He(this,arguments,function(e){1!==this.nodeType&&11!==this.nodeType&&9!==this.nodeType||je(this,e).appendChild(e)})},prepend:function(){return He(this,arguments,function(e){if(1===this.nodeType||11===this.nodeType||9===this.nodeType){var t=je(this,e);t.insertBefore(e,t.firstChild)}})},before:function(){return He(this,arguments,function(e){this.parentNode&&this.parentNode.insertBefore(e,this)})},after:function(){return He(this,arguments,function(e){this.parentNode&&this.parentNode.insertBefore(e,this.nextSibling)})},empty:function(){for(var e,t=0;null!=(e=this[t]);t++)1===e.nodeType&&(S.cleanData(ve(e,!1)),e.textContent="");return this},clone:function(e,t){return e=null!=e&&e,t=null==t?e:t,this.map(function(){return S.clone(this,e,t)})},html:function(e){return $(this,function(e){var t=this[0]||{},n=0,r=this.length;if(void 0===e&&1===t.nodeType)return t.innerHTML;if("string"==typeof e&&!ke.test(e)&&!ge[(de.exec(e)||["",""])[1].toLowerCase()]){e=S.htmlPrefilter(e);try{for(;n<r;n++)1===(t=this[n]||{}).nodeType&&(S.cleanData(ve(t,!1)),t.innerHTML=e);t=0}catch(e){}}t&&this.empty().append(e)},null,e,arguments.length)},replaceWith:function(){var n=[];return He(this,arguments,function(e){var t=this.parentNode;S.inArray(this,n)<0&&(S.cleanData(ve(this)),t&&t.replaceChild(e,this))},n)}}),S.each({appendTo:"append",prependTo:"prepend",insertBefore:"before",insertAfter:"after",replaceAll:"replaceWith"},function(e,a){S.fn[e]=function(e){for(var t,n=[],r=S(e),i=r.length-1,o=0;o<=i;o++)t=o===i?this:this.clone(!0),S(r[o])[a](t),u.apply(n,t.get());return this.pushStack(n)}});var Pe=new RegExp("^("+ee+")(?!px)[a-z%]+$","i"),Re=function(e){var t=e.ownerDocument.defaultView;return t&&t.opener||(t=C),t.getComputedStyle(e)},Me=function(e,t,n){var r,i,o={};for(i in t)o[i]=e.style[i],e.style[i]=t[i];for(i in r=n.call(e),t)e.style[i]=o[i];return r},Ie=new RegExp(ne.join("|"),"i");function We(e,t,n){var r,i,o,a,s=e.style;return(n=n||Re(e))&&(""!==(a=n.getPropertyValue(t)||n[t])||ie(e)||(a=S.style(e,t)),!y.pixelBoxStyles()&&Pe.test(a)&&Ie.test(t)&&(r=s.width,i=s.minWidth,o=s.maxWidth,s.minWidth=s.maxWidth=s.width=a,a=n.width,s.width=r,s.minWidth=i,s.maxWidth=o)),void 0!==a?a+"":a}function Fe(e,t){return{get:function(){if(!e())return(this.get=t).apply(this,arguments);delete this.get}}}!function(){function e(){if(l){u.style.cssText="position:absolute;left:-11111px;width:60px;margin-top:1px;padding:0;border:0",l.style.cssText="position:relative;display:block;box-sizing:border-box;overflow:scroll;margin:auto;border:1px;padding:1px;width:60%;top:1%",re.appendChild(u).appendChild(l);var e=C.getComputedStyle(l);n="1%"!==e.top,s=12===t(e.marginLeft),l.style.right="60%",o=36===t(e.right),r=36===t(e.width),l.style.position="absolute",i=12===t(l.offsetWidth/3),re.removeChild(u),l=null}}function t(e){return Math.round(parseFloat(e))}var n,r,i,o,a,s,u=E.createElement("div"),l=E.createElement("div");l.style&&(l.style.backgroundClip="content-box",l.cloneNode(!0).style.backgroundClip="",y.clearCloneStyle="content-box"===l.style.backgroundClip,S.extend(y,{boxSizingReliable:function(){return e(),r},pixelBoxStyles:function(){return e(),o},pixelPosition:function(){return e(),n},reliableMarginLeft:function(){return e(),s},scrollboxSize:function(){return e(),i},reliableTrDimensions:function(){var e,t,n,r;return null==a&&(e=E.createElement("table"),t=E.createElement("tr"),n=E.createElement("div"),e.style.cssText="position:absolute;left:-11111px;border-collapse:separate",t.style.cssText="border:1px solid",t.style.height="1px",n.style.height="9px",n.style.display="block",re.appendChild(e).appendChild(t).appendChild(n),r=C.getComputedStyle(t),a=parseInt(r.height,10)+parseInt(r.borderTopWidth,10)+parseInt(r.borderBottomWidth,10)===t.offsetHeight,re.removeChild(e)),a}}))}();var Be=["Webkit","Moz","ms"],$e=E.createElement("div").style,_e={};function ze(e){var t=S.cssProps[e]||_e[e];return t||(e in $e?e:_e[e]=function(e){var t=e[0].toUpperCase()+e.slice(1),n=Be.length;while(n--)if((e=Be[n]+t)in $e)return e}(e)||e)}var Ue=/^(none|table(?!-c[ea]).+)/,Xe=/^--/,Ve={position:"absolute",visibility:"hidden",display:"block"},Ge={letterSpacing:"0",fontWeight:"400"};function Ye(e,t,n){var r=te.exec(t);return r?Math.max(0,r[2]-(n||0))+(r[3]||"px"):t}function Qe(e,t,n,r,i,o){var a="width"===t?1:0,s=0,u=0;if(n===(r?"border":"content"))return 0;for(;a<4;a+=2)"margin"===n&&(u+=S.css(e,n+ne[a],!0,i)),r?("content"===n&&(u-=S.css(e,"padding"+ne[a],!0,i)),"margin"!==n&&(u-=S.css(e,"border"+ne[a]+"Width",!0,i))):(u+=S.css(e,"padding"+ne[a],!0,i),"padding"!==n?u+=S.css(e,"border"+ne[a]+"Width",!0,i):s+=S.css(e,"border"+ne[a]+"Width",!0,i));return!r&&0<=o&&(u+=Math.max(0,Math.ceil(e["offset"+t[0].toUpperCase()+t.slice(1)]-o-u-s-.5))||0),u}function Je(e,t,n){var r=Re(e),i=(!y.boxSizingReliable()||n)&&"border-box"===S.css(e,"boxSizing",!1,r),o=i,a=We(e,t,r),s="offset"+t[0].toUpperCase()+t.slice(1);if(Pe.test(a)){if(!n)return a;a="auto"}return(!y.boxSizingReliable()&&i||!y.reliableTrDimensions()&&A(e,"tr")||"auto"===a||!parseFloat(a)&&"inline"===S.css(e,"display",!1,r))&&e.getClientRects().length&&(i="border-box"===S.css(e,"boxSizing",!1,r),(o=s in e)&&(a=e[s])),(a=parseFloat(a)||0)+Qe(e,t,n||(i?"border":"content"),o,r,a)+"px"}function Ke(e,t,n,r,i){return new Ke.prototype.init(e,t,n,r,i)}S.extend({cssHooks:{opacity:{get:function(e,t){if(t){var n=We(e,"opacity");return""===n?"1":n}}}},cssNumber:{animationIterationCount:!0,columnCount:!0,fillOpacity:!0,flexGrow:!0,flexShrink:!0,fontWeight:!0,gridArea:!0,gridColumn:!0,gridColumnEnd:!0,gridColumnStart:!0,gridRow:!0,gridRowEnd:!0,gridRowStart:!0,lineHeight:!0,opacity:!0,order:!0,orphans:!0,widows:!0,zIndex:!0,zoom:!0},cssProps:{},style:function(e,t,n,r){if(e&&3!==e.nodeType&&8!==e.nodeType&&e.style){var i,o,a,s=X(t),u=Xe.test(t),l=e.style;if(u||(t=ze(s)),a=S.cssHooks[t]||S.cssHooks[s],void 0===n)return a&&"get"in a&&void 0!==(i=a.get(e,!1,r))?i:l[t];"string"===(o=typeof n)&&(i=te.exec(n))&&i[1]&&(n=se(e,t,i),o="number"),null!=n&&n==n&&("number"!==o||u||(n+=i&&i[3]||(S.cssNumber[s]?"":"px")),y.clearCloneStyle||""!==n||0!==t.indexOf("background")||(l[t]="inherit"),a&&"set"in a&&void 0===(n=a.set(e,n,r))||(u?l.setProperty(t,n):l[t]=n))}},css:function(e,t,n,r){var i,o,a,s=X(t);return Xe.test(t)||(t=ze(s)),(a=S.cssHooks[t]||S.cssHooks[s])&&"get"in a&&(i=a.get(e,!0,n)),void 0===i&&(i=We(e,t,r)),"normal"===i&&t in Ge&&(i=Ge[t]),""===n||n?(o=parseFloat(i),!0===n||isFinite(o)?o||0:i):i}}),S.each(["height","width"],function(e,u){S.cssHooks[u]={get:function(e,t,n){if(t)return!Ue.test(S.css(e,"display"))||e.getClientRects().length&&e.getBoundingClientRect().width?Je(e,u,n):Me(e,Ve,function(){return Je(e,u,n)})},set:function(e,t,n){var r,i=Re(e),o=!y.scrollboxSize()&&"absolute"===i.position,a=(o||n)&&"border-box"===S.css(e,"boxSizing",!1,i),s=n?Qe(e,u,n,a,i):0;return a&&o&&(s-=Math.ceil(e["offset"+u[0].toUpperCase()+u.slice(1)]-parseFloat(i[u])-Qe(e,u,"border",!1,i)-.5)),s&&(r=te.exec(t))&&"px"!==(r[3]||"px")&&(e.style[u]=t,t=S.css(e,u)),Ye(0,t,s)}}}),S.cssHooks.marginLeft=Fe(y.reliableMarginLeft,function(e,t){if(t)return(parseFloat(We(e,"marginLeft"))||e.getBoundingClientRect().left-Me(e,{marginLeft:0},function(){return e.getBoundingClientRect().left}))+"px"}),S.each({margin:"",padding:"",border:"Width"},function(i,o){S.cssHooks[i+o]={expand:function(e){for(var t=0,n={},r="string"==typeof e?e.split(" "):[e];t<4;t++)n[i+ne[t]+o]=r[t]||r[t-2]||r[0];return n}},"margin"!==i&&(S.cssHooks[i+o].set=Ye)}),S.fn.extend({css:function(e,t){return $(this,function(e,t,n){var r,i,o={},a=0;if(Array.isArray(t)){for(r=Re(e),i=t.length;a<i;a++)o[t[a]]=S.css(e,t[a],!1,r);return o}return void 0!==n?S.style(e,t,n):S.css(e,t)},e,t,1<arguments.length)}}),((S.Tween=Ke).prototype={constructor:Ke,init:function(e,t,n,r,i,o){this.elem=e,this.prop=n,this.easing=i||S.easing._default,this.options=t,this.start=this.now=this.cur(),this.end=r,this.unit=o||(S.cssNumber[n]?"":"px")},cur:function(){var e=Ke.propHooks[this.prop];return e&&e.get?e.get(this):Ke.propHooks._default.get(this)},run:function(e){var t,n=Ke.propHooks[this.prop];return this.options.duration?this.pos=t=S.easing[this.easing](e,this.options.duration*e,0,1,this.options.duration):this.pos=t=e,this.now=(this.end-this.start)*t+this.start,this.options.step&&this.options.step.call(this.elem,this.now,this),n&&n.set?n.set(this):Ke.propHooks._default.set(this),this}}).init.prototype=Ke.prototype,(Ke.propHooks={_default:{get:function(e){var t;return 1!==e.elem.nodeType||null!=e.elem[e.prop]&&null==e.elem.style[e.prop]?e.elem[e.prop]:(t=S.css(e.elem,e.prop,""))&&"auto"!==t?t:0},set:function(e){S.fx.step[e.prop]?S.fx.step[e.prop](e):1!==e.elem.nodeType||!S.cssHooks[e.prop]&&null==e.elem.style[ze(e.prop)]?e.elem[e.prop]=e.now:S.style(e.elem,e.prop,e.now+e.unit)}}}).scrollTop=Ke.propHooks.scrollLeft={set:function(e){e.elem.nodeType&&e.elem.parentNode&&(e.elem[e.prop]=e.now)}},S.easing={linear:function(e){return e},swing:function(e){return.5-Math.cos(e*Math.PI)/2},_default:"swing"},S.fx=Ke.prototype.init,S.fx.step={};var Ze,et,tt,nt,rt=/^(?:toggle|show|hide)$/,it=/queueHooks$/;function ot(){et&&(!1===E.hidden&&C.requestAnimationFrame?C.requestAnimationFrame(ot):C.setTimeout(ot,S.fx.interval),S.fx.tick())}function at(){return C.setTimeout(function(){Ze=void 0}),Ze=Date.now()}function st(e,t){var n,r=0,i={height:e};for(t=t?1:0;r<4;r+=2-t)i["margin"+(n=ne[r])]=i["padding"+n]=e;return t&&(i.opacity=i.width=e),i}function ut(e,t,n){for(var r,i=(lt.tweeners[t]||[]).concat(lt.tweeners["*"]),o=0,a=i.length;o<a;o++)if(r=i[o].call(n,t,e))return r}function lt(o,e,t){var n,a,r=0,i=lt.prefilters.length,s=S.Deferred().always(function(){delete u.elem}),u=function(){if(a)return!1;for(var e=Ze||at(),t=Math.max(0,l.startTime+l.duration-e),n=1-(t/l.duration||0),r=0,i=l.tweens.length;r<i;r++)l.tweens[r].run(n);return s.notifyWith(o,[l,n,t]),n<1&&i?t:(i||s.notifyWith(o,[l,1,0]),s.resolveWith(o,[l]),!1)},l=s.promise({elem:o,props:S.extend({},e),opts:S.extend(!0,{specialEasing:{},easing:S.easing._default},t),originalProperties:e,originalOptions:t,startTime:Ze||at(),duration:t.duration,tweens:[],createTween:function(e,t){var n=S.Tween(o,l.opts,e,t,l.opts.specialEasing[e]||l.opts.easing);return l.tweens.push(n),n},stop:function(e){var t=0,n=e?l.tweens.length:0;if(a)return this;for(a=!0;t<n;t++)l.tweens[t].run(1);return e?(s.notifyWith(o,[l,1,0]),s.resolveWith(o,[l,e])):s.rejectWith(o,[l,e]),this}}),c=l.props;for(!function(e,t){var n,r,i,o,a;for(n in e)if(i=t[r=X(n)],o=e[n],Array.isArray(o)&&(i=o[1],o=e[n]=o[0]),n!==r&&(e[r]=o,delete e[n]),(a=S.cssHooks[r])&&"expand"in a)for(n in o=a.expand(o),delete e[r],o)n in e||(e[n]=o[n],t[n]=i);else t[r]=i}(c,l.opts.specialEasing);r<i;r++)if(n=lt.prefilters[r].call(l,o,c,l.opts))return m(n.stop)&&(S._queueHooks(l.elem,l.opts.queue).stop=n.stop.bind(n)),n;return S.map(c,ut,l),m(l.opts.start)&&l.opts.start.call(o,l),l.progress(l.opts.progress).done(l.opts.done,l.opts.complete).fail(l.opts.fail).always(l.opts.always),S.fx.timer(S.extend(u,{elem:o,anim:l,queue:l.opts.queue})),l}S.Animation=S.extend(lt,{tweeners:{"*":[function(e,t){var n=this.createTween(e,t);return se(n.elem,e,te.exec(t),n),n}]},tweener:function(e,t){m(e)?(t=e,e=["*"]):e=e.match(P);for(var n,r=0,i=e.length;r<i;r++)n=e[r],lt.tweeners[n]=lt.tweeners[n]||[],lt.tweeners[n].unshift(t)},prefilters:[function(e,t,n){var r,i,o,a,s,u,l,c,f="width"in t||"height"in t,p=this,d={},h=e.style,g=e.nodeType&&ae(e),v=Y.get(e,"fxshow");for(r in n.queue||(null==(a=S._queueHooks(e,"fx")).unqueued&&(a.unqueued=0,s=a.empty.fire,a.empty.fire=function(){a.unqueued||s()}),a.unqueued++,p.always(function(){p.always(function(){a.unqueued--,S.queue(e,"fx").length||a.empty.fire()})})),t)if(i=t[r],rt.test(i)){if(delete t[r],o=o||"toggle"===i,i===(g?"hide":"show")){if("show"!==i||!v||void 0===v[r])continue;g=!0}d[r]=v&&v[r]||S.style(e,r)}if((u=!S.isEmptyObject(t))||!S.isEmptyObject(d))for(r in f&&1===e.nodeType&&(n.overflow=[h.overflow,h.overflowX,h.overflowY],null==(l=v&&v.display)&&(l=Y.get(e,"display")),"none"===(c=S.css(e,"display"))&&(l?c=l:(le([e],!0),l=e.style.display||l,c=S.css(e,"display"),le([e]))),("inline"===c||"inline-block"===c&&null!=l)&&"none"===S.css(e,"float")&&(u||(p.done(function(){h.display=l}),null==l&&(c=h.display,l="none"===c?"":c)),h.display="inline-block")),n.overflow&&(h.overflow="hidden",p.always(function(){h.overflow=n.overflow[0],h.overflowX=n.overflow[1],h.overflowY=n.overflow[2]})),u=!1,d)u||(v?"hidden"in v&&(g=v.hidden):v=Y.access(e,"fxshow",{display:l}),o&&(v.hidden=!g),g&&le([e],!0),p.done(function(){for(r in g||le([e]),Y.remove(e,"fxshow"),d)S.style(e,r,d[r])})),u=ut(g?v[r]:0,r,p),r in v||(v[r]=u.start,g&&(u.end=u.start,u.start=0))}],prefilter:function(e,t){t?lt.prefilters.unshift(e):lt.prefilters.push(e)}}),S.speed=function(e,t,n){var r=e&&"object"==typeof e?S.extend({},e):{complete:n||!n&&t||m(e)&&e,duration:e,easing:n&&t||t&&!m(t)&&t};return S.fx.off?r.duration=0:"number"!=typeof r.duration&&(r.duration in S.fx.speeds?r.duration=S.fx.speeds[r.duration]:r.duration=S.fx.speeds._default),null!=r.queue&&!0!==r.queue||(r.queue="fx"),r.old=r.complete,r.complete=function(){m(r.old)&&r.old.call(this),r.queue&&S.dequeue(this,r.queue)},r},S.fn.extend({fadeTo:function(e,t,n,r){return this.filter(ae).css("opacity",0).show().end().animate({opacity:t},e,n,r)},animate:function(t,e,n,r){var i=S.isEmptyObject(t),o=S.speed(e,n,r),a=function(){var e=lt(this,S.extend({},t),o);(i||Y.get(this,"finish"))&&e.stop(!0)};return a.finish=a,i||!1===o.queue?this.each(a):this.queue(o.queue,a)},stop:function(i,e,o){var a=function(e){var t=e.stop;delete e.stop,t(o)};return"string"!=typeof i&&(o=e,e=i,i=void 0),e&&this.queue(i||"fx",[]),this.each(function(){var e=!0,t=null!=i&&i+"queueHooks",n=S.timers,r=Y.get(this);if(t)r[t]&&r[t].stop&&a(r[t]);else for(t in r)r[t]&&r[t].stop&&it.test(t)&&a(r[t]);for(t=n.length;t--;)n[t].elem!==this||null!=i&&n[t].queue!==i||(n[t].anim.stop(o),e=!1,n.splice(t,1));!e&&o||S.dequeue(this,i)})},finish:function(a){return!1!==a&&(a=a||"fx"),this.each(function(){var e,t=Y.get(this),n=t[a+"queue"],r=t[a+"queueHooks"],i=S.timers,o=n?n.length:0;for(t.finish=!0,S.queue(this,a,[]),r&&r.stop&&r.stop.call(this,!0),e=i.length;e--;)i[e].elem===this&&i[e].queue===a&&(i[e].anim.stop(!0),i.splice(e,1));for(e=0;e<o;e++)n[e]&&n[e].finish&&n[e].finish.call(this);delete t.finish})}}),S.each(["toggle","show","hide"],function(e,r){var i=S.fn[r];S.fn[r]=function(e,t,n){return null==e||"boolean"==typeof e?i.apply(this,arguments):this.animate(st(r,!0),e,t,n)}}),S.each({slideDown:st("show"),slideUp:st("hide"),slideToggle:st("toggle"),fadeIn:{opacity:"show"},fadeOut:{opacity:"hide"},fadeToggle:{opacity:"toggle"}},function(e,r){S.fn[e]=function(e,t,n){return this.animate(r,e,t,n)}}),S.timers=[],S.fx.tick=function(){var e,t=0,n=S.timers;for(Ze=Date.now();t<n.length;t++)(e=n[t])()||n[t]!==e||n.splice(t--,1);n.length||S.fx.stop(),Ze=void 0},S.fx.timer=function(e){S.timers.push(e),S.fx.start()},S.fx.interval=13,S.fx.start=function(){et||(et=!0,ot())},S.fx.stop=function(){et=null},S.fx.speeds={slow:600,fast:200,_default:400},S.fn.delay=function(r,e){return r=S.fx&&S.fx.speeds[r]||r,e=e||"fx",this.queue(e,function(e,t){var n=C.setTimeout(e,r);t.stop=function(){C.clearTimeout(n)}})},tt=E.createElement("input"),nt=E.createElement("select").appendChild(E.createElement("option")),tt.type="checkbox",y.checkOn=""!==tt.value,y.optSelected=nt.selected,(tt=E.createElement("input")).value="t",tt.type="radio",y.radioValue="t"===tt.value;var ct,ft=S.expr.attrHandle;S.fn.extend({attr:function(e,t){return $(this,S.attr,e,t,1<arguments.length)},removeAttr:function(e){return this.each(function(){S.removeAttr(this,e)})}}),S.extend({attr:function(e,t,n){var r,i,o=e.nodeType;if(3!==o&&8!==o&&2!==o)return"undefined"==typeof e.getAttribute?S.prop(e,t,n):(1===o&&S.isXMLDoc(e)||(i=S.attrHooks[t.toLowerCase()]||(S.expr.match.bool.test(t)?ct:void 0)),void 0!==n?null===n?void S.removeAttr(e,t):i&&"set"in i&&void 0!==(r=i.set(e,n,t))?r:(e.setAttribute(t,n+""),n):i&&"get"in i&&null!==(r=i.get(e,t))?r:null==(r=S.find.attr(e,t))?void 0:r)},attrHooks:{type:{set:function(e,t){if(!y.radioValue&&"radio"===t&&A(e,"input")){var n=e.value;return e.setAttribute("type",t),n&&(e.value=n),t}}}},removeAttr:function(e,t){var n,r=0,i=t&&t.match(P);if(i&&1===e.nodeType)while(n=i[r++])e.removeAttribute(n)}}),ct={set:function(e,t,n){return!1===t?S.removeAttr(e,n):e.setAttribute(n,n),n}},S.each(S.expr.match.bool.source.match(/\w+/g),function(e,t){var a=ft[t]||S.find.attr;ft[t]=function(e,t,n){var r,i,o=t.toLowerCase();return n||(i=ft[o],ft[o]=r,r=null!=a(e,t,n)?o:null,ft[o]=i),r}});var pt=/^(?:input|select|textarea|button)$/i,dt=/^(?:a|area)$/i;function ht(e){return(e.match(P)||[]).join(" ")}function gt(e){return e.getAttribute&&e.getAttribute("class")||""}function vt(e){return Array.isArray(e)?e:"string"==typeof e&&e.match(P)||[]}S.fn.extend({prop:function(e,t){return $(this,S.prop,e,t,1<arguments.length)},removeProp:function(e){return this.each(function(){delete this[S.propFix[e]||e]})}}),S.extend({prop:function(e,t,n){var r,i,o=e.nodeType;if(3!==o&&8!==o&&2!==o)return 1===o&&S.isXMLDoc(e)||(t=S.propFix[t]||t,i=S.propHooks[t]),void 0!==n?i&&"set"in i&&void 0!==(r=i.set(e,n,t))?r:e[t]=n:i&&"get"in i&&null!==(r=i.get(e,t))?r:e[t]},propHooks:{tabIndex:{get:function(e){var t=S.find.attr(e,"tabindex");return t?parseInt(t,10):pt.test(e.nodeName)||dt.test(e.nodeName)&&e.href?0:-1}}},propFix:{"for":"htmlFor","class":"className"}}),y.optSelected||(S.propHooks.selected={get:function(e){var t=e.parentNode;return t&&t.parentNode&&t.parentNode.selectedIndex,null},set:function(e){var t=e.parentNode;t&&(t.selectedIndex,t.parentNode&&t.parentNode.selectedIndex)}}),S.each(["tabIndex","readOnly","maxLength","cellSpacing","cellPadding","rowSpan","colSpan","useMap","frameBorder","contentEditable"],function(){S.propFix[this.toLowerCase()]=this}),S.fn.extend({addClass:function(t){var e,n,r,i,o,a,s,u=0;if(m(t))return this.each(function(e){S(this).addClass(t.call(this,e,gt(this)))});if((e=vt(t)).length)while(n=this[u++])if(i=gt(n),r=1===n.nodeType&&" "+ht(i)+" "){a=0;while(o=e[a++])r.indexOf(" "+o+" ")<0&&(r+=o+" ");i!==(s=ht(r))&&n.setAttribute("class",s)}return this},removeClass:function(t){var e,n,r,i,o,a,s,u=0;if(m(t))return this.each(function(e){S(this).removeClass(t.call(this,e,gt(this)))});if(!arguments.length)return this.attr("class","");if((e=vt(t)).length)while(n=this[u++])if(i=gt(n),r=1===n.nodeType&&" "+ht(i)+" "){a=0;while(o=e[a++])while(-1<r.indexOf(" "+o+" "))r=r.replace(" "+o+" "," ");i!==(s=ht(r))&&n.setAttribute("class",s)}return this},toggleClass:function(i,t){var o=typeof i,a="string"===o||Array.isArray(i);return"boolean"==typeof t&&a?t?this.addClass(i):this.removeClass(i):m(i)?this.each(function(e){S(this).toggleClass(i.call(this,e,gt(this),t),t)}):this.each(function(){var e,t,n,r;if(a){t=0,n=S(this),r=vt(i);while(e=r[t++])n.hasClass(e)?n.removeClass(e):n.addClass(e)}else void 0!==i&&"boolean"!==o||((e=gt(this))&&Y.set(this,"__className__",e),this.setAttribute&&this.setAttribute("class",e||!1===i?"":Y.get(this,"__className__")||""))})},hasClass:function(e){var t,n,r=0;t=" "+e+" ";while(n=this[r++])if(1===n.nodeType&&-1<(" "+ht(gt(n))+" ").indexOf(t))return!0;return!1}});var yt=/\r/g;S.fn.extend({val:function(n){var r,e,i,t=this[0];return arguments.length?(i=m(n),this.each(function(e){var t;1===this.nodeType&&(null==(t=i?n.call(this,e,S(this).val()):n)?t="":"number"==typeof t?t+="":Array.isArray(t)&&(t=S.map(t,function(e){return null==e?"":e+""})),(r=S.valHooks[this.type]||S.valHooks[this.nodeName.toLowerCase()])&&"set"in r&&void 0!==r.set(this,t,"value")||(this.value=t))})):t?(r=S.valHooks[t.type]||S.valHooks[t.nodeName.toLowerCase()])&&"get"in r&&void 0!==(e=r.get(t,"value"))?e:"string"==typeof(e=t.value)?e.replace(yt,""):null==e?"":e:void 0}}),S.extend({valHooks:{option:{get:function(e){var t=S.find.attr(e,"value");return null!=t?t:ht(S.text(e))}},select:{get:function(e){var t,n,r,i=e.options,o=e.selectedIndex,a="select-one"===e.type,s=a?null:[],u=a?o+1:i.length;for(r=o<0?u:a?o:0;r<u;r++)if(((n=i[r]).selected||r===o)&&!n.disabled&&(!n.parentNode.disabled||!A(n.parentNode,"optgroup"))){if(t=S(n).val(),a)return t;s.push(t)}return s},set:function(e,t){var n,r,i=e.options,o=S.makeArray(t),a=i.length;while(a--)((r=i[a]).selected=-1<S.inArray(S.valHooks.option.get(r),o))&&(n=!0);return n||(e.selectedIndex=-1),o}}}}),S.each(["radio","checkbox"],function(){S.valHooks[this]={set:function(e,t){if(Array.isArray(t))return e.checked=-1<S.inArray(S(e).val(),t)}},y.checkOn||(S.valHooks[this].get=function(e){return null===e.getAttribute("value")?"on":e.value})}),y.focusin="onfocusin"in C;var mt=/^(?:focusinfocus|focusoutblur)$/,xt=function(e){e.stopPropagation()};S.extend(S.event,{trigger:function(e,t,n,r){var i,o,a,s,u,l,c,f,p=[n||E],d=v.call(e,"type")?e.type:e,h=v.call(e,"namespace")?e.namespace.split("."):[];if(o=f=a=n=n||E,3!==n.nodeType&&8!==n.nodeType&&!mt.test(d+S.event.triggered)&&(-1<d.indexOf(".")&&(d=(h=d.split(".")).shift(),h.sort()),u=d.indexOf(":")<0&&"on"+d,(e=e[S.expando]?e:new S.Event(d,"object"==typeof e&&e)).isTrigger=r?2:3,e.namespace=h.join("."),e.rnamespace=e.namespace?new RegExp("(^|\\.)"+h.join("\\.(?:.*\\.|)")+"(\\.|$)"):null,e.result=void 0,e.target||(e.target=n),t=null==t?[e]:S.makeArray(t,[e]),c=S.event.special[d]||{},r||!c.trigger||!1!==c.trigger.apply(n,t))){if(!r&&!c.noBubble&&!x(n)){for(s=c.delegateType||d,mt.test(s+d)||(o=o.parentNode);o;o=o.parentNode)p.push(o),a=o;a===(n.ownerDocument||E)&&p.push(a.defaultView||a.parentWindow||C)}i=0;while((o=p[i++])&&!e.isPropagationStopped())f=o,e.type=1<i?s:c.bindType||d,(l=(Y.get(o,"events")||Object.create(null))[e.type]&&Y.get(o,"handle"))&&l.apply(o,t),(l=u&&o[u])&&l.apply&&V(o)&&(e.result=l.apply(o,t),!1===e.result&&e.preventDefault());return e.type=d,r||e.isDefaultPrevented()||c._default&&!1!==c._default.apply(p.pop(),t)||!V(n)||u&&m(n[d])&&!x(n)&&((a=n[u])&&(n[u]=null),S.event.triggered=d,e.isPropagationStopped()&&f.addEventListener(d,xt),n[d](),e.isPropagationStopped()&&f.removeEventListener(d,xt),S.event.triggered=void 0,a&&(n[u]=a)),e.result}},simulate:function(e,t,n){var r=S.extend(new S.Event,n,{type:e,isSimulated:!0});S.event.trigger(r,null,t)}}),S.fn.extend({trigger:function(e,t){return this.each(function(){S.event.trigger(e,t,this)})},triggerHandler:function(e,t){var n=this[0];if(n)return S.event.trigger(e,t,n,!0)}}),y.focusin||S.each({focus:"focusin",blur:"focusout"},function(n,r){var i=function(e){S.event.simulate(r,e.target,S.event.fix(e))};S.event.special[r]={setup:function(){var e=this.ownerDocument||this.document||this,t=Y.access(e,r);t||e.addEventListener(n,i,!0),Y.access(e,r,(t||0)+1)},teardown:function(){var e=this.ownerDocument||this.document||this,t=Y.access(e,r)-1;t?Y.access(e,r,t):(e.removeEventListener(n,i,!0),Y.remove(e,r))}}});var bt=C.location,wt={guid:Date.now()},Tt=/\?/;S.parseXML=function(e){var t,n;if(!e||"string"!=typeof e)return null;try{t=(new C.DOMParser).parseFromString(e,"text/xml")}catch(e){}return n=t&&t.getElementsByTagName("parsererror")[0],t&&!n||S.error("Invalid XML: "+(n?S.map(n.childNodes,function(e){return e.textContent}).join("\n"):e)),t};var Ct=/\[\]$/,Et=/\r?\n/g,St=/^(?:submit|button|image|reset|file)$/i,kt=/^(?:input|select|textarea|keygen)/i;function At(n,e,r,i){var t;if(Array.isArray(e))S.each(e,function(e,t){r||Ct.test(n)?i(n,t):At(n+"["+("object"==typeof t&&null!=t?e:"")+"]",t,r,i)});else if(r||"object"!==w(e))i(n,e);else for(t in e)At(n+"["+t+"]",e[t],r,i)}S.param=function(e,t){var n,r=[],i=function(e,t){var n=m(t)?t():t;r[r.length]=encodeURIComponent(e)+"="+encodeURIComponent(null==n?"":n)};if(null==e)return"";if(Array.isArray(e)||e.jquery&&!S.isPlainObject(e))S.each(e,function(){i(this.name,this.value)});else for(n in e)At(n,e[n],t,i);return r.join("&")},S.fn.extend({serialize:function(){return S.param(this.serializeArray())},serializeArray:function(){return this.map(function(){var e=S.prop(this,"elements");return e?S.makeArray(e):this}).filter(function(){var e=this.type;return this.name&&!S(this).is(":disabled")&&kt.test(this.nodeName)&&!St.test(e)&&(this.checked||!pe.test(e))}).map(function(e,t){var n=S(this).val();return null==n?null:Array.isArray(n)?S.map(n,function(e){return{name:t.name,value:e.replace(Et,"\r\n")}}):{name:t.name,value:n.replace(Et,"\r\n")}}).get()}});var Nt=/%20/g,jt=/#.*$/,Dt=/([?&])_=[^&]*/,qt=/^(.*?):[ \t]*([^\r\n]*)$/gm,Lt=/^(?:GET|HEAD)$/,Ht=/^\/\//,Ot={},Pt={},Rt="*/".concat("*"),Mt=E.createElement("a");function It(o){return function(e,t){"string"!=typeof e&&(t=e,e="*");var n,r=0,i=e.toLowerCase().match(P)||[];if(m(t))while(n=i[r++])"+"===n[0]?(n=n.slice(1)||"*",(o[n]=o[n]||[]).unshift(t)):(o[n]=o[n]||[]).push(t)}}function Wt(t,i,o,a){var s={},u=t===Pt;function l(e){var r;return s[e]=!0,S.each(t[e]||[],function(e,t){var n=t(i,o,a);return"string"!=typeof n||u||s[n]?u?!(r=n):void 0:(i.dataTypes.unshift(n),l(n),!1)}),r}return l(i.dataTypes[0])||!s["*"]&&l("*")}function Ft(e,t){var n,r,i=S.ajaxSettings.flatOptions||{};for(n in t)void 0!==t[n]&&((i[n]?e:r||(r={}))[n]=t[n]);return r&&S.extend(!0,e,r),e}Mt.href=bt.href,S.extend({active:0,lastModified:{},etag:{},ajaxSettings:{url:bt.href,type:"GET",isLocal:/^(?:about|app|app-storage|.+-extension|file|res|widget):$/.test(bt.protocol),global:!0,processData:!0,async:!0,contentType:"application/x-www-form-urlencoded; charset=UTF-8",accepts:{"*":Rt,text:"text/plain",html:"text/html",xml:"application/xml, text/xml",json:"application/json, text/javascript"},contents:{xml:/\bxml\b/,html:/\bhtml/,json:/\bjson\b/},responseFields:{xml:"responseXML",text:"responseText",json:"responseJSON"},converters:{"* text":String,"text html":!0,"text json":JSON.parse,"text xml":S.parseXML},flatOptions:{url:!0,context:!0}},ajaxSetup:function(e,t){return t?Ft(Ft(e,S.ajaxSettings),t):Ft(S.ajaxSettings,e)},ajaxPrefilter:It(Ot),ajaxTransport:It(Pt),ajax:function(e,t){"object"==typeof e&&(t=e,e=void 0),t=t||{};var c,f,p,n,d,r,h,g,i,o,v=S.ajaxSetup({},t),y=v.context||v,m=v.context&&(y.nodeType||y.jquery)?S(y):S.event,x=S.Deferred(),b=S.Callbacks("once memory"),w=v.statusCode||{},a={},s={},u="canceled",T={readyState:0,getResponseHeader:function(e){var t;if(h){if(!n){n={};while(t=qt.exec(p))n[t[1].toLowerCase()+" "]=(n[t[1].toLowerCase()+" "]||[]).concat(t[2])}t=n[e.toLowerCase()+" "]}return null==t?null:t.join(", ")},getAllResponseHeaders:function(){return h?p:null},setRequestHeader:function(e,t){return null==h&&(e=s[e.toLowerCase()]=s[e.toLowerCase()]||e,a[e]=t),this},overrideMimeType:function(e){return null==h&&(v.mimeType=e),this},statusCode:function(e){var t;if(e)if(h)T.always(e[T.status]);else for(t in e)w[t]=[w[t],e[t]];return this},abort:function(e){var t=e||u;return c&&c.abort(t),l(0,t),this}};if(x.promise(T),v.url=((e||v.url||bt.href)+"").replace(Ht,bt.protocol+"//"),v.type=t.method||t.type||v.method||v.type,v.dataTypes=(v.dataType||"*").toLowerCase().match(P)||[""],null==v.crossDomain){r=E.createElement("a");try{r.href=v.url,r.href=r.href,v.crossDomain=Mt.protocol+"//"+Mt.host!=r.protocol+"//"+r.host}catch(e){v.crossDomain=!0}}if(v.data&&v.processData&&"string"!=typeof v.data&&(v.data=S.param(v.data,v.traditional)),Wt(Ot,v,t,T),h)return T;for(i in(g=S.event&&v.global)&&0==S.active++&&S.event.trigger("ajaxStart"),v.type=v.type.toUpperCase(),v.hasContent=!Lt.test(v.type),f=v.url.replace(jt,""),v.hasContent?v.data&&v.processData&&0===(v.contentType||"").indexOf("application/x-www-form-urlencoded")&&(v.data=v.data.replace(Nt,"+")):(o=v.url.slice(f.length),v.data&&(v.processData||"string"==typeof v.data)&&(f+=(Tt.test(f)?"&":"?")+v.data,delete v.data),!1===v.cache&&(f=f.replace(Dt,"$1"),o=(Tt.test(f)?"&":"?")+"_="+wt.guid+++o),v.url=f+o),v.ifModified&&(S.lastModified[f]&&T.setRequestHeader("If-Modified-Since",S.lastModified[f]),S.etag[f]&&T.setRequestHeader("If-None-Match",S.etag[f])),(v.data&&v.hasContent&&!1!==v.contentType||t.contentType)&&T.setRequestHeader("Content-Type",v.contentType),T.setRequestHeader("Accept",v.dataTypes[0]&&v.accepts[v.dataTypes[0]]?v.accepts[v.dataTypes[0]]+("*"!==v.dataTypes[0]?", "+Rt+"; q=0.01":""):v.accepts["*"]),v.headers)T.setRequestHeader(i,v.headers[i]);if(v.beforeSend&&(!1===v.beforeSend.call(y,T,v)||h))return T.abort();if(u="abort",b.add(v.complete),T.done(v.success),T.fail(v.error),c=Wt(Pt,v,t,T)){if(T.readyState=1,g&&m.trigger("ajaxSend",[T,v]),h)return T;v.async&&0<v.timeout&&(d=C.setTimeout(function(){T.abort("timeout")},v.timeout));try{h=!1,c.send(a,l)}catch(e){if(h)throw e;l(-1,e)}}else l(-1,"No Transport");function l(e,t,n,r){var i,o,a,s,u,l=t;h||(h=!0,d&&C.clearTimeout(d),c=void 0,p=r||"",T.readyState=0<e?4:0,i=200<=e&&e<300||304===e,n&&(s=function(e,t,n){var r,i,o,a,s=e.contents,u=e.dataTypes;while("*"===u[0])u.shift(),void 0===r&&(r=e.mimeType||t.getResponseHeader("Content-Type"));if(r)for(i in s)if(s[i]&&s[i].test(r)){u.unshift(i);break}if(u[0]in n)o=u[0];else{for(i in n){if(!u[0]||e.converters[i+" "+u[0]]){o=i;break}a||(a=i)}o=o||a}if(o)return o!==u[0]&&u.unshift(o),n[o]}(v,T,n)),!i&&-1<S.inArray("script",v.dataTypes)&&S.inArray("json",v.dataTypes)<0&&(v.converters["text script"]=function(){}),s=function(e,t,n,r){var i,o,a,s,u,l={},c=e.dataTypes.slice();if(c[1])for(a in e.converters)l[a.toLowerCase()]=e.converters[a];o=c.shift();while(o)if(e.responseFields[o]&&(n[e.responseFields[o]]=t),!u&&r&&e.dataFilter&&(t=e.dataFilter(t,e.dataType)),u=o,o=c.shift())if("*"===o)o=u;else if("*"!==u&&u!==o){if(!(a=l[u+" "+o]||l["* "+o]))for(i in l)if((s=i.split(" "))[1]===o&&(a=l[u+" "+s[0]]||l["* "+s[0]])){!0===a?a=l[i]:!0!==l[i]&&(o=s[0],c.unshift(s[1]));break}if(!0!==a)if(a&&e["throws"])t=a(t);else try{t=a(t)}catch(e){return{state:"parsererror",error:a?e:"No conversion from "+u+" to "+o}}}return{state:"success",data:t}}(v,s,T,i),i?(v.ifModified&&((u=T.getResponseHeader("Last-Modified"))&&(S.lastModified[f]=u),(u=T.getResponseHeader("etag"))&&(S.etag[f]=u)),204===e||"HEAD"===v.type?l="nocontent":304===e?l="notmodified":(l=s.state,o=s.data,i=!(a=s.error))):(a=l,!e&&l||(l="error",e<0&&(e=0))),T.status=e,T.statusText=(t||l)+"",i?x.resolveWith(y,[o,l,T]):x.rejectWith(y,[T,l,a]),T.statusCode(w),w=void 0,g&&m.trigger(i?"ajaxSuccess":"ajaxError",[T,v,i?o:a]),b.fireWith(y,[T,l]),g&&(m.trigger("ajaxComplete",[T,v]),--S.active||S.event.trigger("ajaxStop")))}return T},getJSON:function(e,t,n){return S.get(e,t,n,"json")},getScript:function(e,t){return S.get(e,void 0,t,"script")}}),S.each(["get","post"],function(e,i){S[i]=function(e,t,n,r){return m(t)&&(r=r||n,n=t,t=void 0),S.ajax(S.extend({url:e,type:i,dataType:r,data:t,success:n},S.isPlainObject(e)&&e))}}),S.ajaxPrefilter(function(e){var t;for(t in e.headers)"content-type"===t.toLowerCase()&&(e.contentType=e.headers[t]||"")}),S._evalUrl=function(e,t,n){return S.ajax({url:e,type:"GET",dataType:"script",cache:!0,async:!1,global:!1,converters:{"text script":function(){}},dataFilter:function(e){S.globalEval(e,t,n)}})},S.fn.extend({wrapAll:function(e){var t;return this[0]&&(m(e)&&(e=e.call(this[0])),t=S(e,this[0].ownerDocument).eq(0).clone(!0),this[0].parentNode&&t.insertBefore(this[0]),t.map(function(){var e=this;while(e.firstElementChild)e=e.firstElementChild;return e}).append(this)),this},wrapInner:function(n){return m(n)?this.each(function(e){S(this).wrapInner(n.call(this,e))}):this.each(function(){var e=S(this),t=e.contents();t.length?t.wrapAll(n):e.append(n)})},wrap:function(t){var n=m(t);return this.each(function(e){S(this).wrapAll(n?t.call(this,e):t)})},unwrap:function(e){return this.parent(e).not("body").each(function(){S(this).replaceWith(this.childNodes)}),this}}),S.expr.pseudos.hidden=function(e){return!S.expr.pseudos.visible(e)},S.expr.pseudos.visible=function(e){return!!(e.offsetWidth||e.offsetHeight||e.getClientRects().length)},S.ajaxSettings.xhr=function(){try{return new C.XMLHttpRequest}catch(e){}};var Bt={0:200,1223:204},$t=S.ajaxSettings.xhr();y.cors=!!$t&&"withCredentials"in $t,y.ajax=$t=!!$t,S.ajaxTransport(function(i){var o,a;if(y.cors||$t&&!i.crossDomain)return{send:function(e,t){var n,r=i.xhr();if(r.open(i.type,i.url,i.async,i.username,i.password),i.xhrFields)for(n in i.xhrFields)r[n]=i.xhrFields[n];for(n in i.mimeType&&r.overrideMimeType&&r.overrideMimeType(i.mimeType),i.crossDomain||e["X-Requested-With"]||(e["X-Requested-With"]="XMLHttpRequest"),e)r.setRequestHeader(n,e[n]);o=function(e){return function(){o&&(o=a=r.onload=r.onerror=r.onabort=r.ontimeout=r.onreadystatechange=null,"abort"===e?r.abort():"error"===e?"number"!=typeof r.status?t(0,"error"):t(r.status,r.statusText):t(Bt[r.status]||r.status,r.statusText,"text"!==(r.responseType||"text")||"string"!=typeof r.responseText?{binary:r.response}:{text:r.responseText},r.getAllResponseHeaders()))}},r.onload=o(),a=r.onerror=r.ontimeout=o("error"),void 0!==r.onabort?r.onabort=a:r.onreadystatechange=function(){4===r.readyState&&C.setTimeout(function(){o&&a()})},o=o("abort");try{r.send(i.hasContent&&i.data||null)}catch(e){if(o)throw e}},abort:function(){o&&o()}}}),S.ajaxPrefilter(function(e){e.crossDomain&&(e.contents.script=!1)}),S.ajaxSetup({accepts:{script:"text/javascript, application/javascript, application/ecmascript, application/x-ecmascript"},contents:{script:/\b(?:java|ecma)script\b/},converters:{"text script":function(e){return S.globalEval(e),e}}}),S.ajaxPrefilter("script",function(e){void 0===e.cache&&(e.cache=!1),e.crossDomain&&(e.type="GET")}),S.ajaxTransport("script",function(n){var r,i;if(n.crossDomain||n.scriptAttrs)return{send:function(e,t){r=S("<script>").attr(n.scriptAttrs||{}).prop({charset:n.scriptCharset,src:n.url}).on("load error",i=function(e){r.remove(),i=null,e&&t("error"===e.type?404:200,e.type)}),E.head.appendChild(r[0])},abort:function(){i&&i()}}});var _t,zt=[],Ut=/(=)\?(?=&|$)|\?\?/;S.ajaxSetup({jsonp:"callback",jsonpCallback:function(){var e=zt.pop()||S.expando+"_"+wt.guid++;return this[e]=!0,e}}),S.ajaxPrefilter("json jsonp",function(e,t,n){var r,i,o,a=!1!==e.jsonp&&(Ut.test(e.url)?"url":"string"==typeof e.data&&0===(e.contentType||"").indexOf("application/x-www-form-urlencoded")&&Ut.test(e.data)&&"data");if(a||"jsonp"===e.dataTypes[0])return r=e.jsonpCallback=m(e.jsonpCallback)?e.jsonpCallback():e.jsonpCallback,a?e[a]=e[a].replace(Ut,"$1"+r):!1!==e.jsonp&&(e.url+=(Tt.test(e.url)?"&":"?")+e.jsonp+"="+r),e.converters["script json"]=function(){return o||S.error(r+" was not called"),o[0]},e.dataTypes[0]="json",i=C[r],C[r]=function(){o=arguments},n.always(function(){void 0===i?S(C).removeProp(r):C[r]=i,e[r]&&(e.jsonpCallback=t.jsonpCallback,zt.push(r)),o&&m(i)&&i(o[0]),o=i=void 0}),"script"}),y.createHTMLDocument=((_t=E.implementation.createHTMLDocument("").body).innerHTML="<form></form><form></form>",2===_t.childNodes.length),S.parseHTML=function(e,t,n){return"string"!=typeof e?[]:("boolean"==typeof t&&(n=t,t=!1),t||(y.createHTMLDocument?((r=(t=E.implementation.createHTMLDocument("")).createElement("base")).href=E.location.href,t.head.appendChild(r)):t=E),o=!n&&[],(i=N.exec(e))?[t.createElement(i[1])]:(i=xe([e],t,o),o&&o.length&&S(o).remove(),S.merge([],i.childNodes)));var r,i,o},S.fn.load=function(e,t,n){var r,i,o,a=this,s=e.indexOf(" ");return-1<s&&(r=ht(e.slice(s)),e=e.slice(0,s)),m(t)?(n=t,t=void 0):t&&"object"==typeof t&&(i="POST"),0<a.length&&S.ajax({url:e,type:i||"GET",dataType:"html",data:t}).done(function(e){o=arguments,a.html(r?S("<div>").append(S.parseHTML(e)).find(r):e)}).always(n&&function(e,t){a.each(function(){n.apply(this,o||[e.responseText,t,e])})}),this},S.expr.pseudos.animated=function(t){return S.grep(S.timers,function(e){return t===e.elem}).length},S.offset={setOffset:function(e,t,n){var r,i,o,a,s,u,l=S.css(e,"position"),c=S(e),f={};"static"===l&&(e.style.position="relative"),s=c.offset(),o=S.css(e,"top"),u=S.css(e,"left"),("absolute"===l||"fixed"===l)&&-1<(o+u).indexOf("auto")?(a=(r=c.position()).top,i=r.left):(a=parseFloat(o)||0,i=parseFloat(u)||0),m(t)&&(t=t.call(e,n,S.extend({},s))),null!=t.top&&(f.top=t.top-s.top+a),null!=t.left&&(f.left=t.left-s.left+i),"using"in t?t.using.call(e,f):c.css(f)}},S.fn.extend({offset:function(t){if(arguments.length)return void 0===t?this:this.each(function(e){S.offset.setOffset(this,t,e)});var e,n,r=this[0];return r?r.getClientRects().length?(e=r.getBoundingClientRect(),n=r.ownerDocument.defaultView,{top:e.top+n.pageYOffset,left:e.left+n.pageXOffset}):{top:0,left:0}:void 0},position:function(){if(this[0]){var e,t,n,r=this[0],i={top:0,left:0};if("fixed"===S.css(r,"position"))t=r.getBoundingClientRect();else{t=this.offset(),n=r.ownerDocument,e=r.offsetParent||n.documentElement;while(e&&(e===n.body||e===n.documentElement)&&"static"===S.css(e,"position"))e=e.parentNode;e&&e!==r&&1===e.nodeType&&((i=S(e).offset()).top+=S.css(e,"borderTopWidth",!0),i.left+=S.css(e,"borderLeftWidth",!0))}return{top:t.top-i.top-S.css(r,"marginTop",!0),left:t.left-i.left-S.css(r,"marginLeft",!0)}}},offsetParent:function(){return this.map(function(){var e=this.offsetParent;while(e&&"static"===S.css(e,"position"))e=e.offsetParent;return e||re})}}),S.each({scrollLeft:"pageXOffset",scrollTop:"pageYOffset"},function(t,i){var o="pageYOffset"===i;S.fn[t]=function(e){return $(this,function(e,t,n){var r;if(x(e)?r=e:9===e.nodeType&&(r=e.defaultView),void 0===n)return r?r[i]:e[t];r?r.scrollTo(o?r.pageXOffset:n,o?n:r.pageYOffset):e[t]=n},t,e,arguments.length)}}),S.each(["top","left"],function(e,n){S.cssHooks[n]=Fe(y.pixelPosition,function(e,t){if(t)return t=We(e,n),Pe.test(t)?S(e).position()[n]+"px":t})}),S.each({Height:"height",Width:"width"},function(a,s){S.each({padding:"inner"+a,content:s,"":"outer"+a},function(r,o){S.fn[o]=function(e,t){var n=arguments.length&&(r||"boolean"!=typeof e),i=r||(!0===e||!0===t?"margin":"border");return $(this,function(e,t,n){var r;return x(e)?0===o.indexOf("outer")?e["inner"+a]:e.document.documentElement["client"+a]:9===e.nodeType?(r=e.documentElement,Math.max(e.body["scroll"+a],r["scroll"+a],e.body["offset"+a],r["offset"+a],r["client"+a])):void 0===n?S.css(e,t,i):S.style(e,t,n,i)},s,n?e:void 0,n)}})}),S.each(["ajaxStart","ajaxStop","ajaxComplete","ajaxError","ajaxSuccess","ajaxSend"],function(e,t){S.fn[t]=function(e){return this.on(t,e)}}),S.fn.extend({bind:function(e,t,n){return this.on(e,null,t,n)},unbind:function(e,t){return this.off(e,null,t)},delegate:function(e,t,n,r){return this.on(t,e,n,r)},undelegate:function(e,t,n){return 1===arguments.length?this.off(e,"**"):this.off(t,e||"**",n)},hover:function(e,t){return this.mouseenter(e).mouseleave(t||e)}}),S.each("blur focus focusin focusout resize scroll click dblclick mousedown mouseup mousemove mouseover mouseout mouseenter mouseleave change select submit keydown keypress keyup contextmenu".split(" "),function(e,n){S.fn[n]=function(e,t){return 0<arguments.length?this.on(n,null,e,t):this.trigger(n)}});var Xt=/^[\s\uFEFF\xA0]+|[\s\uFEFF\xA0]+$/g;S.proxy=function(e,t){var n,r,i;if("string"==typeof t&&(n=e[t],t=e,e=n),m(e))return r=s.call(arguments,2),(i=function(){return e.apply(t||this,r.concat(s.call(arguments)))}).guid=e.guid=e.guid||S.guid++,i},S.holdReady=function(e){e?S.readyWait++:S.ready(!0)},S.isArray=Array.isArray,S.parseJSON=JSON.parse,S.nodeName=A,S.isFunction=m,S.isWindow=x,S.camelCase=X,S.type=w,S.now=Date.now,S.isNumeric=function(e){var t=S.type(e);return("number"===t||"string"===t)&&!isNaN(e-parseFloat(e))},S.trim=function(e){return null==e?"":(e+"").replace(Xt,"")},"function"==typeof define&&define.amd&&define("jquery",[],function(){return S});var Vt=C.jQuery,Gt=C.$;return S.noConflict=function(e){return C.$===S&&(C.$=Gt),e&&C.jQuery===S&&(C.jQuery=Vt),S},"undefined"==typeof e&&(C.jQuery=C.$=S),S});
</script>
<style type="text/css">.leaflet-pane,.leaflet-tile,.leaflet-marker-icon,.leaflet-marker-shadow,.leaflet-tile-container,.leaflet-pane > svg,.leaflet-pane > canvas,.leaflet-zoom-box,.leaflet-image-layer,.leaflet-layer {position: absolute;left: 0;top: 0;}.leaflet-container {overflow: hidden;}.leaflet-tile,.leaflet-marker-icon,.leaflet-marker-shadow {-webkit-user-select: none;-moz-user-select: none;user-select: none;-webkit-user-drag: none;}.leaflet-safari .leaflet-tile {image-rendering: -webkit-optimize-contrast;}.leaflet-safari .leaflet-tile-container {width: 1600px;height: 1600px;-webkit-transform-origin: 0 0;}.leaflet-marker-icon,.leaflet-marker-shadow {display: block;}.leaflet-container .leaflet-overlay-pane svg,.leaflet-container .leaflet-marker-pane img,.leaflet-container .leaflet-shadow-pane img,.leaflet-container .leaflet-tile-pane img,.leaflet-container img.leaflet-image-layer {max-width: none !important;max-height: none !important;}.leaflet-container.leaflet-touch-zoom {-ms-touch-action: pan-x pan-y;touch-action: pan-x pan-y;}.leaflet-container.leaflet-touch-drag {-ms-touch-action: pinch-zoom;touch-action: none;touch-action: pinch-zoom;}.leaflet-container.leaflet-touch-drag.leaflet-touch-zoom {-ms-touch-action: none;touch-action: none;}.leaflet-container {-webkit-tap-highlight-color: transparent;}.leaflet-container a {-webkit-tap-highlight-color: rgba(51, 181, 229, 0.4);}.leaflet-tile {filter: inherit;visibility: hidden;}.leaflet-tile-loaded {visibility: inherit;}.leaflet-zoom-box {width: 0;height: 0;-moz-box-sizing: border-box;box-sizing: border-box;z-index: 800;}.leaflet-overlay-pane svg {-moz-user-select: none;}.leaflet-pane { z-index: 400; }.leaflet-tile-pane { z-index: 200; }.leaflet-overlay-pane { z-index: 400; }.leaflet-shadow-pane { z-index: 500; }.leaflet-marker-pane { z-index: 600; }.leaflet-tooltip-pane { z-index: 650; }.leaflet-popup-pane { z-index: 700; }.leaflet-map-pane canvas { z-index: 100; }.leaflet-map-pane svg { z-index: 200; }.leaflet-vml-shape {width: 1px;height: 1px;}.lvml {behavior: url(#default#VML);display: inline-block;position: absolute;}.leaflet-control {position: relative;z-index: 800;pointer-events: visiblePainted; pointer-events: auto;}.leaflet-top,.leaflet-bottom {position: absolute;z-index: 1000;pointer-events: none;}.leaflet-top {top: 0;}.leaflet-right {right: 0;}.leaflet-bottom {bottom: 0;}.leaflet-left {left: 0;}.leaflet-control {float: left;clear: both;}.leaflet-right .leaflet-control {float: right;}.leaflet-top .leaflet-control {margin-top: 10px;}.leaflet-bottom .leaflet-control {margin-bottom: 10px;}.leaflet-left .leaflet-control {margin-left: 10px;}.leaflet-right .leaflet-control {margin-right: 10px;}.leaflet-fade-anim .leaflet-tile {will-change: opacity;}.leaflet-fade-anim .leaflet-popup {opacity: 0;-webkit-transition: opacity 0.2s linear;-moz-transition: opacity 0.2s linear;-o-transition: opacity 0.2s linear;transition: opacity 0.2s linear;}.leaflet-fade-anim .leaflet-map-pane .leaflet-popup {opacity: 1;}.leaflet-zoom-animated {-webkit-transform-origin: 0 0;-ms-transform-origin: 0 0;transform-origin: 0 0;}.leaflet-zoom-anim .leaflet-zoom-animated {will-change: transform;}.leaflet-zoom-anim .leaflet-zoom-animated {-webkit-transition: -webkit-transform 0.25s cubic-bezier(0,0,0.25,1);-moz-transition: -moz-transform 0.25s cubic-bezier(0,0,0.25,1);-o-transition: -o-transform 0.25s cubic-bezier(0,0,0.25,1);transition: transform 0.25s cubic-bezier(0,0,0.25,1);}.leaflet-zoom-anim .leaflet-tile,.leaflet-pan-anim .leaflet-tile {-webkit-transition: none;-moz-transition: none;-o-transition: none;transition: none;}.leaflet-zoom-anim .leaflet-zoom-hide {visibility: hidden;}.leaflet-interactive {cursor: pointer;}.leaflet-grab {cursor: -webkit-grab;cursor: -moz-grab;}.leaflet-crosshair,.leaflet-crosshair .leaflet-interactive {cursor: crosshair;}.leaflet-popup-pane,.leaflet-control {cursor: auto;}.leaflet-dragging .leaflet-grab,.leaflet-dragging .leaflet-grab .leaflet-interactive,.leaflet-dragging .leaflet-marker-draggable {cursor: move;cursor: -webkit-grabbing;cursor: -moz-grabbing;}.leaflet-marker-icon,.leaflet-marker-shadow,.leaflet-image-layer,.leaflet-pane > svg path,.leaflet-tile-container {pointer-events: none;}.leaflet-marker-icon.leaflet-interactive,.leaflet-image-layer.leaflet-interactive,.leaflet-pane > svg path.leaflet-interactive {pointer-events: visiblePainted; pointer-events: auto;}.leaflet-container {background: #ddd;outline: 0;}.leaflet-container a {color: #0078A8;}.leaflet-container a.leaflet-active {outline: 2px solid orange;}.leaflet-zoom-box {border: 2px dotted #38f;background: rgba(255,255,255,0.5);}.leaflet-container {font: 12px/1.5 "Helvetica Neue", Arial, Helvetica, sans-serif;}.leaflet-bar {box-shadow: 0 1px 5px rgba(0,0,0,0.65);border-radius: 4px;}.leaflet-bar a,.leaflet-bar a:hover {background-color: #fff;border-bottom: 1px solid #ccc;width: 26px;height: 26px;line-height: 26px;display: block;text-align: center;text-decoration: none;color: black;}.leaflet-bar a,.leaflet-control-layers-toggle {background-position: 50% 50%;background-repeat: no-repeat;display: block;}.leaflet-bar a:hover {background-color: #f4f4f4;}.leaflet-bar a:first-child {border-top-left-radius: 4px;border-top-right-radius: 4px;}.leaflet-bar a:last-child {border-bottom-left-radius: 4px;border-bottom-right-radius: 4px;border-bottom: none;}.leaflet-bar a.leaflet-disabled {cursor: default;background-color: #f4f4f4;color: #bbb;}.leaflet-touch .leaflet-bar a {width: 30px;height: 30px;line-height: 30px;}.leaflet-touch .leaflet-bar a:first-child {border-top-left-radius: 2px;border-top-right-radius: 2px;}.leaflet-touch .leaflet-bar a:last-child {border-bottom-left-radius: 2px;border-bottom-right-radius: 2px;}.leaflet-control-zoom-in,.leaflet-control-zoom-out {font: bold 18px 'Lucida Console', Monaco, monospace;text-indent: 1px;}.leaflet-touch .leaflet-control-zoom-in, .leaflet-touch .leaflet-control-zoom-out {font-size: 22px;}.leaflet-control-layers {box-shadow: 0 1px 5px rgba(0,0,0,0.4);background: #fff;border-radius: 5px;}.leaflet-control-layers-toggle {background-image: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABoAAAAaCAQAAAADQ4RFAAACf0lEQVR4AY1UM3gkARTePdvdoTxXKc+qTl3aU5U6b2Kbkz3Gtq3Zw6ziLGNPzrYx7946Tr6/ee/XeCQ4D3ykPtL5tHno4n0d/h3+xfuWHGLX81cn7r0iTNzjr7LrlxCqPtkbTQEHeqOrTy4Yyt3VCi/IOB0v7rVC7q45Q3Gr5K6jt+3Gl5nCoDD4MtO+j96Wu8atmhGqcNGHObuf8OM/x3AMx38+4Z2sPqzCxRFK2aF2e5Jol56XTLyggAMTL56XOMoS1W4pOyjUcGGQdZxU6qRh7B9Zp+PfpOFlqt0zyDZckPi1ttmIp03jX8gyJ8a/PG2yutpS/Vol7peZIbZcKBAEEheEIAgFbDkz5H6Zrkm2hVWGiXKiF4Ycw0RWKdtC16Q7qe3X4iOMxruonzegJzWaXFrU9utOSsLUmrc0YjeWYjCW4PDMADElpJSSQ0vQvA1Tm6/JlKnqFs1EGyZiFCqnRZTEJJJiKRYzVYzJck2Rm6P4iH+cmSY0YzimYa8l0EtTODFWhcMIMVqdsI2uiTvKmTisIDHJ3od5GILVhBCarCfVRmo4uTjkhrhzkiBV7SsaqS+TzrzM1qpGGUFt28pIySQHR6h7F6KSwGWm97ay+Z+ZqMcEjEWebE7wxCSQwpkhJqoZA5ivCdZDjJepuJ9IQjGGUmuXJdBFUygxVqVsxFsLMbDe8ZbDYVCGKxs+W080max1hFCarCfV+C1KATwcnvE9gRRuMP2prdbWGowm1KB1y+zwMMENkM755cJ2yPDtqhTI6ED1M/82yIDtC/4j4BijjeObflpO9I9MwXTCsSX8jWAFeHr05WoLTJ5G8IQVS/7vwR6ohirYM7f6HzYpogfS3R2OAAAAAElFTkSuQmCC);width: 36px;height: 36px;}.leaflet-retina .leaflet-control-layers-toggle {background-image: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAADQAAAA0CAQAAABvcdNgAAAEsklEQVR4AWL4TydIhpZK1kpWOlg0w3ZXP6D2soBtG42jeI6ZmQTHzAxiTbSJsYLjO9HhP+WOmcuhciVnmHVQcJnp7DFvScowZorad/+V/fVzMdMT2g9Cv9guXGv/7pYOrXh2U+RRR3dSd9JRx6bIFc/ekqHI29JC6pJ5ZEh1yWkhkbcFeSjxgx3L2m1cb1C7bceyxA+CNjT/Ifff+/kDk2u/w/33/IeCMOSaWZ4glosqT3DNnNZQ7Cs58/3Ce5HL78iZH/vKVIaYlqzfdLu8Vi7dnvUbEza5Idt36tquZFldl6N5Z/POLof0XLK61mZCmJSWjVF9tEjUluu74IUXvgttuVIHE7YxSkaYhJZam7yiM9Pv82JYfl9nptxZaxMJE4YSPty+vF0+Y2up9d3wwijfjZbabqm/3bZ9ecKHsiGmRflnn1MW4pjHf9oLufyn2z3y1D6n8g8TZhxyzipLNPnAUpsOiuWimg52psrTZYnOWYNDTMuWBWa0tJb4rgq1UvmutpaYEbZlwU3CLJm/ayYjHW5/h7xWLn9Hh1vepDkyf7dE7MtT5LR4e7yYpHrkhOUpEfssBLq2pPhAqoSWKUkk7EDqkmK6RrCEzqDjhNDWNE+XSMvkJRDWlZTmCW0l0PHQGRZY5t1L83kT0Y3l2SItk5JAWHl2dCOBm+fPu3fo5/3v61RMCO9Jx2EEYYhb0rmNQMX/vm7gqOEJLcXTGw3CAuRNeyaPWwjR8PRqKQ1PDA/dpv+on9Shox52WFnx0KY8onHayrJzm87i5h9xGw/tfkev0jGsQizqezUKjk12hBMKJ4kbCqGPVNXudyyrShovGw5CgxsRICxF6aRmSjlBnHRzg7Gx8fKqEubI2rahQYdR1YgDIRQO7JvQyD52hoIQx0mxa0ODtW2Iozn1le2iIRdzwWewedyZzewidueOGqlsn1MvcnQpuVwLGG3/IR1hIKxCjelIDZ8ldqWz25jWAsnldEnK0Zxro19TGVb2ffIZEsIO89EIEDvKMPrzmBOQcKQ+rroye6NgRRxqR4U8EAkz0CL6uSGOm6KQCdWjvjRiSP1BPalCRS5iQYiEIvxuBMJEWgzSoHADcVMuN7IuqqTeyUPq22qFimFtxDyBBJEwNyt6TM88blFHao/6tWWhuuOM4SAK4EI4QmFHA+SEyWlp4EQoJ13cYGzMu7yszEIBOm2rVmHUNqwAIQabISNMRstmdhNWcFLsSm+0tjJH1MdRxO5Nx0WDMhCtgD6OKgZeljJqJKc9po8juskR9XN0Y1lZ3mWjLR9JCO1jRDMd0fpYC2VnvjBSEFg7wBENc0R9HFlb0xvF1+TBEpF68d+DHR6IOWVv2BECtxo46hOFUBd/APU57WIoEwJhIi2CdpyZX0m93BZicktMj1AS9dClteUFAUNUIEygRZCtik5zSxI9MubTBH1GOiHsiLJ3OCoSZkILa9PxiN0EbvhsAo8tdAf9Seepd36lGWHmtNANTv5Jd0z4QYyeo/UEJqxKRpg5LZx6btLPsOaEmdMyxYdlc8LMaJnikDlhclqmPiQnTEpLUIZEwkRagjYkEibQErwhkTAKCLQEbUgkzJQWc/0PstHHcfEdQ+UAAAAASUVORK5CYII=);background-size: 26px 26px;}.leaflet-touch .leaflet-control-layers-toggle {width: 44px;height: 44px;}.leaflet-control-layers .leaflet-control-layers-list,.leaflet-control-layers-expanded .leaflet-control-layers-toggle {display: none;}.leaflet-control-layers-expanded .leaflet-control-layers-list {display: block;position: relative;}.leaflet-control-layers-expanded {padding: 6px 10px 6px 6px;color: #333;background: #fff;}.leaflet-control-layers-scrollbar {overflow-y: scroll;overflow-x: hidden;padding-right: 5px;}.leaflet-control-layers-selector {margin-top: 2px;position: relative;top: 1px;}.leaflet-control-layers label {display: block;}.leaflet-control-layers-separator {height: 0;border-top: 1px solid #ddd;margin: 5px -10px 5px -6px;}.leaflet-default-icon-path {background-image: url(data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABkAAAApCAYAAADAk4LOAAAFgUlEQVR4Aa1XA5BjWRTN2oW17d3YaZtr2962HUzbDNpjszW24mRt28p47v7zq/bXZtrp/lWnXr337j3nPCe85NcypgSFdugCpW5YoDAMRaIMqRi6aKq5E3YqDQO3qAwjVWrD8Ncq/RBpykd8oZUb/kaJutow8r1aP9II0WmLKLIsJyv1w/kqw9Ch2MYdB++12Onxee/QMwvf4/Dk/Lfp/i4nxTXtOoQ4pW5Aj7wpici1A9erdAN2OH64x8OSP9j3Ft3b7aWkTg/Fm91siTra0f9on5sQr9INejH6CUUUpavjFNq1B+Oadhxmnfa8RfEmN8VNAsQhPqF55xHkMzz3jSmChWU6f7/XZKNH+9+hBLOHYozuKQPxyMPUKkrX/K0uWnfFaJGS1QPRtZsOPtr3NsW0uyh6NNCOkU3Yz+bXbT3I8G3xE5EXLXtCXbbqwCO9zPQYPRTZ5vIDXD7U+w7rFDEoUUf7ibHIR4y6bLVPXrz8JVZEql13trxwue/uDivd3fkWRbS6/IA2bID4uk0UpF1N8qLlbBlXs4Ee7HLTfV1j54APvODnSfOWBqtKVvjgLKzF5YdEk5ewRkGlK0i33Eofffc7HT56jD7/6U+qH3Cx7SBLNntH5YIPvODnyfIXZYRVDPqgHtLs5ABHD3YzLuespb7t79FY34DjMwrVrcTuwlT55YMPvOBnRrJ4VXTdNnYug5ucHLBjEpt30701A3Ts+HEa73u6dT3FNWwflY86eMHPk+Yu+i6pzUpRrW7SNDg5JHR4KapmM5Wv2E8Tfcb1HoqqHMHU+uWDD7zg54mz5/2BSnizi9T1Dg4QQXLToGNCkb6tb1NU+QAlGr1++eADrzhn/u8Q2YZhQVlZ5+CAOtqfbhmaUCS1ezNFVm2imDbPmPng5wmz+gwh+oHDce0eUtQ6OGDIyR0uUhUsoO3vfDmmgOezH0mZN59x7MBi++WDL1g/eEiU3avlidO671bkLfwbw5XV2P8Pzo0ydy4t2/0eu33xYSOMOD8hTf4CrBtGMSoXfPLchX+J0ruSePw3LZeK0juPJbYzrhkH0io7B3k164hiGvawhOKMLkrQLyVpZg8rHFW7E2uHOL888IBPlNZ1FPzstSJM694fWr6RwpvcJK60+0HCILTBzZLFNdtAzJaohze60T8qBzyh5ZuOg5e7uwQppofEmf2++DYvmySqGBuKaicF1blQjhuHdvCIMvp8whTTfZzI7RldpwtSzL+F1+wkdZ2TBOW2gIF88PBTzD/gpeREAMEbxnJcaJHNHrpzji0gQCS6hdkEeYt9DF/2qPcEC8RM28Hwmr3sdNyht00byAut2k3gufWNtgtOEOFGUwcXWNDbdNbpgBGxEvKkOQsxivJx33iow0Vw5S6SVTrpVq11ysA2Rp7gTfPfktc6zhtXBBC+adRLshf6sG2RfHPZ5EAc4sVZ83yCN00Fk/4kggu40ZTvIEm5g24qtU4KjBrx/BTTH8ifVASAG7gKrnWxJDcU7x8X6Ecczhm3o6YicvsLXWfh3Ch1W0k8x0nXF+0fFxgt4phz8QvypiwCCFKMqXCnqXExjq10beH+UUA7+nG6mdG/Pu0f3LgFcGrl2s0kNNjpmoJ9o4B29CMO8dMT4Q5ox8uitF6fqsrJOr8qnwNbRzv6hSnG5wP+64C7h9lp30hKNtKdWjtdkbuPA19nJ7Tz3zR/ibgARbhb4AlhavcBebmTHcFl2fvYEnW0ox9xMxKBS8btJ+KiEbq9zA4RthQXDhPa0T9TEe69gWupwc6uBUphquXgf+/FrIjweHQS4/pduMe5ERUMHUd9xv8ZR98CxkS4F2n3EUrUZ10EYNw7BWm9x1GiPssi3GgiGRDKWRYZfXlON+dfNbM+GgIwYdwAAAAASUVORK5CYII=);}.leaflet-container .leaflet-control-attribution {background: #fff;background: rgba(255, 255, 255, 0.7);margin: 0;}.leaflet-control-attribution,.leaflet-control-scale-line {padding: 0 5px;color: #333;}.leaflet-control-attribution a {text-decoration: none;}.leaflet-control-attribution a:hover {text-decoration: underline;}.leaflet-container .leaflet-control-attribution,.leaflet-container .leaflet-control-scale {font-size: 11px;}.leaflet-left .leaflet-control-scale {margin-left: 5px;}.leaflet-bottom .leaflet-control-scale {margin-bottom: 5px;}.leaflet-control-scale-line {border: 2px solid #777;border-top: none;line-height: 1.1;padding: 2px 5px 1px;font-size: 11px;white-space: nowrap;overflow: hidden;-moz-box-sizing: border-box;box-sizing: border-box;background: #fff;background: rgba(255, 255, 255, 0.5);}.leaflet-control-scale-line:not(:first-child) {border-top: 2px solid #777;border-bottom: none;margin-top: -2px;}.leaflet-control-scale-line:not(:first-child):not(:last-child) {border-bottom: 2px solid #777;}.leaflet-touch .leaflet-control-attribution,.leaflet-touch .leaflet-control-layers,.leaflet-touch .leaflet-bar {box-shadow: none;}.leaflet-touch .leaflet-control-layers,.leaflet-touch .leaflet-bar {border: 2px solid rgba(0,0,0,0.2);background-clip: padding-box;}.leaflet-popup {position: absolute;text-align: center;margin-bottom: 20px;}.leaflet-popup-content-wrapper {padding: 1px;text-align: left;border-radius: 12px;}.leaflet-popup-content {margin: 13px 19px;line-height: 1.4;}.leaflet-popup-content p {margin: 18px 0;}.leaflet-popup-tip-container {width: 40px;height: 20px;position: absolute;left: 50%;margin-left: -20px;overflow: hidden;pointer-events: none;}.leaflet-popup-tip {width: 17px;height: 17px;padding: 1px;margin: -10px auto 0;-webkit-transform: rotate(45deg);-moz-transform: rotate(45deg);-ms-transform: rotate(45deg);-o-transform: rotate(45deg);transform: rotate(45deg);}.leaflet-popup-content-wrapper,.leaflet-popup-tip {background: white;color: #333;box-shadow: 0 3px 14px rgba(0,0,0,0.4);}.leaflet-container a.leaflet-popup-close-button {position: absolute;top: 0;right: 0;padding: 4px 4px 0 0;border: none;text-align: center;width: 18px;height: 14px;font: 16px/14px Tahoma, Verdana, sans-serif;color: #c3c3c3;text-decoration: none;font-weight: bold;background: transparent;}.leaflet-container a.leaflet-popup-close-button:hover {color: #999;}.leaflet-popup-scrolled {overflow: auto;border-bottom: 1px solid #ddd;border-top: 1px solid #ddd;}.leaflet-oldie .leaflet-popup-content-wrapper {zoom: 1;}.leaflet-oldie .leaflet-popup-tip {width: 24px;margin: 0 auto;-ms-filter: "progid:DXImageTransform.Microsoft.Matrix(M11=0.70710678, M12=0.70710678, M21=-0.70710678, M22=0.70710678)";filter: progid:DXImageTransform.Microsoft.Matrix(M11=0.70710678, M12=0.70710678, M21=-0.70710678, M22=0.70710678);}.leaflet-oldie .leaflet-popup-tip-container {margin-top: -1px;}.leaflet-oldie .leaflet-control-zoom,.leaflet-oldie .leaflet-control-layers,.leaflet-oldie .leaflet-popup-content-wrapper,.leaflet-oldie .leaflet-popup-tip {border: 1px solid #999;}.leaflet-div-icon {background: #fff;border: 1px solid #666;}.leaflet-tooltip {position: absolute;padding: 6px;background-color: #fff;border: 1px solid #fff;border-radius: 3px;color: #222;white-space: nowrap;-webkit-user-select: none;-moz-user-select: none;-ms-user-select: none;user-select: none;pointer-events: none;box-shadow: 0 1px 3px rgba(0,0,0,0.4);}.leaflet-tooltip.leaflet-clickable {cursor: pointer;pointer-events: auto;}.leaflet-tooltip-top:before,.leaflet-tooltip-bottom:before,.leaflet-tooltip-left:before,.leaflet-tooltip-right:before {position: absolute;pointer-events: none;border: 6px solid transparent;background: transparent;content: "";}.leaflet-tooltip-bottom {margin-top: 6px;}.leaflet-tooltip-top {margin-top: -6px;}.leaflet-tooltip-bottom:before,.leaflet-tooltip-top:before {left: 50%;margin-left: -6px;}.leaflet-tooltip-top:before {bottom: 0;margin-bottom: -12px;border-top-color: #fff;}.leaflet-tooltip-bottom:before {top: 0;margin-top: -12px;margin-left: -6px;border-bottom-color: #fff;}.leaflet-tooltip-left {margin-left: -6px;}.leaflet-tooltip-right {margin-left: 6px;}.leaflet-tooltip-left:before,.leaflet-tooltip-right:before {top: 50%;margin-top: -6px;}.leaflet-tooltip-left:before {right: 0;margin-right: -12px;border-left-color: #fff;}.leaflet-tooltip-right:before {left: 0;margin-left: -12px;border-right-color: #fff;}</style>
<script>/* @preserve
 * Leaflet 1.3.1+Detached: ba6f97fff8647e724e4dfe66d2ed7da11f908989.ba6f97f, a JS library for interactive maps. https://leafletjs.com
 * (c) 2010-2017 Vladimir Agafonkin, (c) 2010-2011 CloudMade
 */
!function(t,i){"object"==typeof exports&&"undefined"!=typeof module?i(exports):"function"==typeof define&&define.amd?define(["exports"],i):i(t.L={})}(this,function(t){"use strict";function i(t){var i,e,n,o;for(e=1,n=arguments.length;e<n;e++){o=arguments[e];for(i in o)t[i]=o[i]}return t}function e(t,i){var e=Array.prototype.slice;if(t.bind)return t.bind.apply(t,e.call(arguments,1));var n=e.call(arguments,2);return function(){return t.apply(i,n.length?n.concat(e.call(arguments)):arguments)}}function n(t){return t._leaflet_id=t._leaflet_id||++ti,t._leaflet_id}function o(t,i,e){var n,o,s,r;return r=function(){n=!1,o&&(s.apply(e,o),o=!1)},s=function(){n?o=arguments:(t.apply(e,arguments),setTimeout(r,i),n=!0)}}function s(t,i,e){var n=i[1],o=i[0],s=n-o;return t===n&&e?t:((t-o)%s+s)%s+o}function r(){return!1}function a(t,i){var e=Math.pow(10,void 0===i?6:i);return Math.round(t*e)/e}function h(t){return t.trim?t.trim():t.replace(/^\s+|\s+$/g,"")}function u(t){return h(t).split(/\s+/)}function l(t,i){t.hasOwnProperty("options")||(t.options=t.options?Qt(t.options):{});for(var e in i)t.options[e]=i[e];return t.options}function c(t,i,e){var n=[];for(var o in t)n.push(encodeURIComponent(e?o.toUpperCase():o)+"="+encodeURIComponent(t[o]));return(i&&-1!==i.indexOf("?")?"&":"?")+n.join("&")}function _(t,i){return t.replace(ii,function(t,e){var n=i[e];if(void 0===n)throw new Error("No value provided for variable "+t);return"function"==typeof n&&(n=n(i)),n})}function d(t,i){for(var e=0;e<t.length;e++)if(t[e]===i)return e;return-1}function p(t){return window["webkit"+t]||window["moz"+t]||window["ms"+t]}function m(t){var i=+new Date,e=Math.max(0,16-(i-oi));return oi=i+e,window.setTimeout(t,e)}function f(t,i,n){if(!n||si!==m)return si.call(window,e(t,i));t.call(i)}function g(t){t&&ri.call(window,t)}function v(){}function y(t){if("undefined"!=typeof L&&L&&L.Mixin){t=ei(t)?t:[t];for(var i=0;i<t.length;i++)t[i]===L.Mixin.Events&&console.warn("Deprecated include of L.Mixin.Events: this property will be removed in future releases, please inherit from L.Evented instead.",(new Error).stack)}}function x(t,i,e){this.x=e?Math.round(t):t,this.y=e?Math.round(i):i}function w(t,i,e){return t instanceof x?t:ei(t)?new x(t[0],t[1]):void 0===t||null===t?t:"object"==typeof t&&"x"in t&&"y"in t?new x(t.x,t.y):new x(t,i,e)}function P(t,i){if(t)for(var e=i?[t,i]:t,n=0,o=e.length;n<o;n++)this.extend(e[n])}function b(t,i){return!t||t instanceof P?t:new P(t,i)}function T(t,i){if(t)for(var e=i?[t,i]:t,n=0,o=e.length;n<o;n++)this.extend(e[n])}function z(t,i){return t instanceof T?t:new T(t,i)}function M(t,i,e){if(isNaN(t)||isNaN(i))throw new Error("Invalid LatLng object: ("+t+", "+i+")");this.lat=+t,this.lng=+i,void 0!==e&&(this.alt=+e)}function C(t,i,e){return t instanceof M?t:ei(t)&&"object"!=typeof t[0]?3===t.length?new M(t[0],t[1],t[2]):2===t.length?new M(t[0],t[1]):null:void 0===t||null===t?t:"object"==typeof t&&"lat"in t?new M(t.lat,"lng"in t?t.lng:t.lon,t.alt):void 0===i?null:new M(t,i,e)}function Z(t,i,e,n){if(ei(t))return this._a=t[0],this._b=t[1],this._c=t[2],void(this._d=t[3]);this._a=t,this._b=i,this._c=e,this._d=n}function S(t,i,e,n){return new Z(t,i,e,n)}function E(t){return document.createElementNS("http://www.w3.org/2000/svg",t)}function k(t,i){var e,n,o,s,r,a,h="";for(e=0,o=t.length;e<o;e++){for(n=0,s=(r=t[e]).length;n<s;n++)a=r[n],h+=(n?"L":"M")+a.x+" "+a.y;h+=i?Xi?"z":"x":""}return h||"M0 0"}function A(t){return navigator.userAgent.toLowerCase().indexOf(t)>=0}function I(t,i,e,n){return"touchstart"===i?O(t,e,n):"touchmove"===i?W(t,e,n):"touchend"===i&&H(t,e,n),this}function B(t,i,e){var n=t["_leaflet_"+i+e];return"touchstart"===i?t.removeEventListener(Qi,n,!1):"touchmove"===i?t.removeEventListener(te,n,!1):"touchend"===i&&(t.removeEventListener(ie,n,!1),t.removeEventListener(ee,n,!1)),this}function O(t,i,n){var o=e(function(t){if("mouse"!==t.pointerType&&t.MSPOINTER_TYPE_MOUSE&&t.pointerType!==t.MSPOINTER_TYPE_MOUSE){if(!(ne.indexOf(t.target.tagName)<0))return;$(t)}j(t,i)});t["_leaflet_touchstart"+n]=o,t.addEventListener(Qi,o,!1),se||(document.documentElement.addEventListener(Qi,R,!0),document.documentElement.addEventListener(te,D,!0),document.documentElement.addEventListener(ie,N,!0),document.documentElement.addEventListener(ee,N,!0),se=!0)}function R(t){oe[t.pointerId]=t,re++}function D(t){oe[t.pointerId]&&(oe[t.pointerId]=t)}function N(t){delete oe[t.pointerId],re--}function j(t,i){t.touches=[];for(var e in oe)t.touches.push(oe[e]);t.changedTouches=[t],i(t)}function W(t,i,e){var n=function(t){(t.pointerType!==t.MSPOINTER_TYPE_MOUSE&&"mouse"!==t.pointerType||0!==t.buttons)&&j(t,i)};t["_leaflet_touchmove"+e]=n,t.addEventListener(te,n,!1)}function H(t,i,e){var n=function(t){j(t,i)};t["_leaflet_touchend"+e]=n,t.addEventListener(ie,n,!1),t.addEventListener(ee,n,!1)}function F(t,i,e){function n(t){var i;if(Ui){if(!Pi||"mouse"===t.pointerType)return;i=re}else i=t.touches.length;if(!(i>1)){var e=Date.now(),n=e-(s||e);r=t.touches?t.touches[0]:t,a=n>0&&n<=h,s=e}}function o(t){if(a&&!r.cancelBubble){if(Ui){if(!Pi||"mouse"===t.pointerType)return;var e,n,o={};for(n in r)e=r[n],o[n]=e&&e.bind?e.bind(r):e;r=o}r.type="dblclick",i(r),s=null}}var s,r,a=!1,h=250;return t[ue+ae+e]=n,t[ue+he+e]=o,t[ue+"dblclick"+e]=i,t.addEventListener(ae,n,!1),t.addEventListener(he,o,!1),t.addEventListener("dblclick",i,!1),this}function U(t,i){var e=t[ue+ae+i],n=t[ue+he+i],o=t[ue+"dblclick"+i];return t.removeEventListener(ae,e,!1),t.removeEventListener(he,n,!1),Pi||t.removeEventListener("dblclick",o,!1),this}function V(t,i,e,n){if("object"==typeof i)for(var o in i)G(t,o,i[o],e);else for(var s=0,r=(i=u(i)).length;s<r;s++)G(t,i[s],e,n);return this}function q(t,i,e,n){if("object"==typeof i)for(var o in i)K(t,o,i[o],e);else if(i)for(var s=0,r=(i=u(i)).length;s<r;s++)K(t,i[s],e,n);else{for(var a in t[le])K(t,a,t[le][a]);delete t[le]}return this}function G(t,i,e,o){var s=i+n(e)+(o?"_"+n(o):"");if(t[le]&&t[le][s])return this;var r=function(i){return e.call(o||t,i||window.event)},a=r;Ui&&0===i.indexOf("touch")?I(t,i,r,s):!Vi||"dblclick"!==i||!F||Ui&&Si?"addEventListener"in t?"mousewheel"===i?t.addEventListener("onwheel"in t?"wheel":"mousewheel",r,!1):"mouseenter"===i||"mouseleave"===i?(r=function(i){i=i||window.event,ot(t,i)&&a(i)},t.addEventListener("mouseenter"===i?"mouseover":"mouseout",r,!1)):("click"===i&&Ti&&(r=function(t){st(t,a)}),t.addEventListener(i,r,!1)):"attachEvent"in t&&t.attachEvent("on"+i,r):F(t,r,s),t[le]=t[le]||{},t[le][s]=r}function K(t,i,e,o){var s=i+n(e)+(o?"_"+n(o):""),r=t[le]&&t[le][s];if(!r)return this;Ui&&0===i.indexOf("touch")?B(t,i,s):!Vi||"dblclick"!==i||!U||Ui&&Si?"removeEventListener"in t?"mousewheel"===i?t.removeEventListener("onwheel"in t?"wheel":"mousewheel",r,!1):t.removeEventListener("mouseenter"===i?"mouseover":"mouseleave"===i?"mouseout":i,r,!1):"detachEvent"in t&&t.detachEvent("on"+i,r):U(t,s),t[le][s]=null}function Y(t){return t.stopPropagation?t.stopPropagation():t.originalEvent?t.originalEvent._stopped=!0:t.cancelBubble=!0,nt(t),this}function X(t){return G(t,"mousewheel",Y),this}function J(t){return V(t,"mousedown touchstart dblclick",Y),G(t,"click",et),this}function $(t){return t.preventDefault?t.preventDefault():t.returnValue=!1,this}function Q(t){return $(t),Y(t),this}function tt(t,i){if(!i)return new x(t.clientX,t.clientY);var e=i.getBoundingClientRect(),n=e.width/i.offsetWidth||1,o=e.height/i.offsetHeight||1;return new x(t.clientX/n-e.left-i.clientLeft,t.clientY/o-e.top-i.clientTop)}function it(t){return Pi?t.wheelDeltaY/2:t.deltaY&&0===t.deltaMode?-t.deltaY/ce:t.deltaY&&1===t.deltaMode?20*-t.deltaY:t.deltaY&&2===t.deltaMode?60*-t.deltaY:t.deltaX||t.deltaZ?0:t.wheelDelta?(t.wheelDeltaY||t.wheelDelta)/2:t.detail&&Math.abs(t.detail)<32765?20*-t.detail:t.detail?t.detail/-32765*60:0}function et(t){_e[t.type]=!0}function nt(t){var i=_e[t.type];return _e[t.type]=!1,i}function ot(t,i){var e=i.relatedTarget;if(!e)return!0;try{for(;e&&e!==t;)e=e.parentNode}catch(t){return!1}return e!==t}function st(t,i){var e=t.timeStamp||t.originalEvent&&t.originalEvent.timeStamp,n=pi&&e-pi;n&&n>100&&n<500||t.target._simulatedClick&&!t._simulated?Q(t):(pi=e,i(t))}function rt(t){return"string"==typeof t?document.getElementById(t):t}function at(t,i){var e=t.style[i]||t.currentStyle&&t.currentStyle[i];if((!e||"auto"===e)&&document.defaultView){var n=document.defaultView.getComputedStyle(t,null);e=n?n[i]:null}return"auto"===e?null:e}function ht(t,i,e){var n=document.createElement(t);return n.className=i||"",e&&e.appendChild(n),n}function ut(t){var i=t.parentNode;i&&i.removeChild(t)}function lt(t){for(;t.firstChild;)t.removeChild(t.firstChild)}function ct(t){var i=t.parentNode;i.lastChild!==t&&i.appendChild(t)}function _t(t){var i=t.parentNode;i.firstChild!==t&&i.insertBefore(t,i.firstChild)}function dt(t,i){if(void 0!==t.classList)return t.classList.contains(i);var e=gt(t);return e.length>0&&new RegExp("(^|\\s)"+i+"(\\s|$)").test(e)}function pt(t,i){if(void 0!==t.classList)for(var e=u(i),n=0,o=e.length;n<o;n++)t.classList.add(e[n]);else if(!dt(t,i)){var s=gt(t);ft(t,(s?s+" ":"")+i)}}function mt(t,i){void 0!==t.classList?t.classList.remove(i):ft(t,h((" "+gt(t)+" ").replace(" "+i+" "," ")))}function ft(t,i){void 0===t.className.baseVal?t.className=i:t.className.baseVal=i}function gt(t){return void 0===t.className.baseVal?t.className:t.className.baseVal}function vt(t,i){"opacity"in t.style?t.style.opacity=i:"filter"in t.style&&yt(t,i)}function yt(t,i){var e=!1,n="DXImageTransform.Microsoft.Alpha";try{e=t.filters.item(n)}catch(t){if(1===i)return}i=Math.round(100*i),e?(e.Enabled=100!==i,e.Opacity=i):t.style.filter+=" progid:"+n+"(opacity="+i+")"}function xt(t){for(var i=document.documentElement.style,e=0;e<t.length;e++)if(t[e]in i)return t[e];return!1}function wt(t,i,e){var n=i||new x(0,0);t.style[pe]=(Oi?"translate("+n.x+"px,"+n.y+"px)":"translate3d("+n.x+"px,"+n.y+"px,0)")+(e?" scale("+e+")":"")}function Lt(t,i){t._leaflet_pos=i,Ni?wt(t,i):(t.style.left=i.x+"px",t.style.top=i.y+"px")}function Pt(t){return t._leaflet_pos||new x(0,0)}function bt(){V(window,"dragstart",$)}function Tt(){q(window,"dragstart",$)}function zt(t){for(;-1===t.tabIndex;)t=t.parentNode;t.style&&(Mt(),ve=t,ye=t.style.outline,t.style.outline="none",V(window,"keydown",Mt))}function Mt(){ve&&(ve.style.outline=ye,ve=void 0,ye=void 0,q(window,"keydown",Mt))}function Ct(t,i){if(!i||!t.length)return t.slice();var e=i*i;return t=kt(t,e),t=St(t,e)}function Zt(t,i,e){return Math.sqrt(Rt(t,i,e,!0))}function St(t,i){var e=t.length,n=new(typeof Uint8Array!=void 0+""?Uint8Array:Array)(e);n[0]=n[e-1]=1,Et(t,n,i,0,e-1);var o,s=[];for(o=0;o<e;o++)n[o]&&s.push(t[o]);return s}function Et(t,i,e,n,o){var s,r,a,h=0;for(r=n+1;r<=o-1;r++)(a=Rt(t[r],t[n],t[o],!0))>h&&(s=r,h=a);h>e&&(i[s]=1,Et(t,i,e,n,s),Et(t,i,e,s,o))}function kt(t,i){for(var e=[t[0]],n=1,o=0,s=t.length;n<s;n++)Ot(t[n],t[o])>i&&(e.push(t[n]),o=n);return o<s-1&&e.push(t[s-1]),e}function At(t,i,e,n,o){var s,r,a,h=n?Se:Bt(t,e),u=Bt(i,e);for(Se=u;;){if(!(h|u))return[t,i];if(h&u)return!1;a=Bt(r=It(t,i,s=h||u,e,o),e),s===h?(t=r,h=a):(i=r,u=a)}}function It(t,i,e,n,o){var s,r,a=i.x-t.x,h=i.y-t.y,u=n.min,l=n.max;return 8&e?(s=t.x+a*(l.y-t.y)/h,r=l.y):4&e?(s=t.x+a*(u.y-t.y)/h,r=u.y):2&e?(s=l.x,r=t.y+h*(l.x-t.x)/a):1&e&&(s=u.x,r=t.y+h*(u.x-t.x)/a),new x(s,r,o)}function Bt(t,i){var e=0;return t.x<i.min.x?e|=1:t.x>i.max.x&&(e|=2),t.y<i.min.y?e|=4:t.y>i.max.y&&(e|=8),e}function Ot(t,i){var e=i.x-t.x,n=i.y-t.y;return e*e+n*n}function Rt(t,i,e,n){var o,s=i.x,r=i.y,a=e.x-s,h=e.y-r,u=a*a+h*h;return u>0&&((o=((t.x-s)*a+(t.y-r)*h)/u)>1?(s=e.x,r=e.y):o>0&&(s+=a*o,r+=h*o)),a=t.x-s,h=t.y-r,n?a*a+h*h:new x(s,r)}function Dt(t){return!ei(t[0])||"object"!=typeof t[0][0]&&void 0!==t[0][0]}function Nt(t){return console.warn("Deprecated use of _flat, please use L.LineUtil.isFlat instead."),Dt(t)}function jt(t,i,e){var n,o,s,r,a,h,u,l,c,_=[1,4,2,8];for(o=0,u=t.length;o<u;o++)t[o]._code=Bt(t[o],i);for(r=0;r<4;r++){for(l=_[r],n=[],o=0,s=(u=t.length)-1;o<u;s=o++)a=t[o],h=t[s],a._code&l?h._code&l||((c=It(h,a,l,i,e))._code=Bt(c,i),n.push(c)):(h._code&l&&((c=It(h,a,l,i,e))._code=Bt(c,i),n.push(c)),n.push(a));t=n}return t}function Wt(t,i){var e,n,o,s,r="Feature"===t.type?t.geometry:t,a=r?r.coordinates:null,h=[],u=i&&i.pointToLayer,l=i&&i.coordsToLatLng||Ht;if(!a&&!r)return null;switch(r.type){case"Point":return e=l(a),u?u(t,e):new Xe(e);case"MultiPoint":for(o=0,s=a.length;o<s;o++)e=l(a[o]),h.push(u?u(t,e):new Xe(e));return new qe(h);case"LineString":case"MultiLineString":return n=Ft(a,"LineString"===r.type?0:1,l),new tn(n,i);case"Polygon":case"MultiPolygon":return n=Ft(a,"Polygon"===r.type?1:2,l),new en(n,i);case"GeometryCollection":for(o=0,s=r.geometries.length;o<s;o++){var c=Wt({geometry:r.geometries[o],type:"Feature",properties:t.properties},i);c&&h.push(c)}return new qe(h);default:throw new Error("Invalid GeoJSON object.")}}function Ht(t){return new M(t[1],t[0],t[2])}function Ft(t,i,e){for(var n,o=[],s=0,r=t.length;s<r;s++)n=i?Ft(t[s],i-1,e):(e||Ht)(t[s]),o.push(n);return o}function Ut(t,i){return i="number"==typeof i?i:6,void 0!==t.alt?[a(t.lng,i),a(t.lat,i),a(t.alt,i)]:[a(t.lng,i),a(t.lat,i)]}function Vt(t,i,e,n){for(var o=[],s=0,r=t.length;s<r;s++)o.push(i?Vt(t[s],i-1,e,n):Ut(t[s],n));return!i&&e&&o.push(o[0]),o}function qt(t,e){return t.feature?i({},t.feature,{geometry:e}):Gt(e)}function Gt(t){return"Feature"===t.type||"FeatureCollection"===t.type?t:{type:"Feature",properties:{},geometry:t}}function Kt(t,i){return new nn(t,i)}function Yt(t,i){return new dn(t,i)}function Xt(t){return Yi?new fn(t):null}function Jt(t){return Xi||Ji?new xn(t):null}var $t=Object.freeze;Object.freeze=function(t){return t};var Qt=Object.create||function(){function t(){}return function(i){return t.prototype=i,new t}}(),ti=0,ii=/\{ *([\w_-]+) *\}/g,ei=Array.isArray||function(t){return"[object Array]"===Object.prototype.toString.call(t)},ni="data:image/gif;base64,R0lGODlhAQABAAD/ACwAAAAAAQABAAACADs=",oi=0,si=window.requestAnimationFrame||p("RequestAnimationFrame")||m,ri=window.cancelAnimationFrame||p("CancelAnimationFrame")||p("CancelRequestAnimationFrame")||function(t){window.clearTimeout(t)},ai=(Object.freeze||Object)({freeze:$t,extend:i,create:Qt,bind:e,lastId:ti,stamp:n,throttle:o,wrapNum:s,falseFn:r,formatNum:a,trim:h,splitWords:u,setOptions:l,getParamString:c,template:_,isArray:ei,indexOf:d,emptyImageUrl:ni,requestFn:si,cancelFn:ri,requestAnimFrame:f,cancelAnimFrame:g});v.extend=function(t){var e=function(){this.initialize&&this.initialize.apply(this,arguments),this.callInitHooks()},n=e.__super__=this.prototype,o=Qt(n);o.constructor=e,e.prototype=o;for(var s in this)this.hasOwnProperty(s)&&"prototype"!==s&&"__super__"!==s&&(e[s]=this[s]);return t.statics&&(i(e,t.statics),delete t.statics),t.includes&&(y(t.includes),i.apply(null,[o].concat(t.includes)),delete t.includes),o.options&&(t.options=i(Qt(o.options),t.options)),i(o,t),o._initHooks=[],o.callInitHooks=function(){if(!this._initHooksCalled){n.callInitHooks&&n.callInitHooks.call(this),this._initHooksCalled=!0;for(var t=0,i=o._initHooks.length;t<i;t++)o._initHooks[t].call(this)}},e},v.include=function(t){return i(this.prototype,t),this},v.mergeOptions=function(t){return i(this.prototype.options,t),this},v.addInitHook=function(t){var i=Array.prototype.slice.call(arguments,1),e="function"==typeof t?t:function(){this[t].apply(this,i)};return this.prototype._initHooks=this.prototype._initHooks||[],this.prototype._initHooks.push(e),this};var hi={on:function(t,i,e){if("object"==typeof t)for(var n in t)this._on(n,t[n],i);else for(var o=0,s=(t=u(t)).length;o<s;o++)this._on(t[o],i,e);return this},off:function(t,i,e){if(t)if("object"==typeof t)for(var n in t)this._off(n,t[n],i);else for(var o=0,s=(t=u(t)).length;o<s;o++)this._off(t[o],i,e);else delete this._events;return this},_on:function(t,i,e){this._events=this._events||{};var n=this._events[t];n||(n=[],this._events[t]=n),e===this&&(e=void 0);for(var o={fn:i,ctx:e},s=n,r=0,a=s.length;r<a;r++)if(s[r].fn===i&&s[r].ctx===e)return;s.push(o)},_off:function(t,i,e){var n,o,s;if(this._events&&(n=this._events[t]))if(i){if(e===this&&(e=void 0),n)for(o=0,s=n.length;o<s;o++){var a=n[o];if(a.ctx===e&&a.fn===i)return a.fn=r,this._firingCount&&(this._events[t]=n=n.slice()),void n.splice(o,1)}}else{for(o=0,s=n.length;o<s;o++)n[o].fn=r;delete this._events[t]}},fire:function(t,e,n){if(!this.listens(t,n))return this;var o=i({},e,{type:t,target:this,sourceTarget:e&&e.sourceTarget||this});if(this._events){var s=this._events[t];if(s){this._firingCount=this._firingCount+1||1;for(var r=0,a=s.length;r<a;r++){var h=s[r];h.fn.call(h.ctx||this,o)}this._firingCount--}}return n&&this._propagateEvent(o),this},listens:function(t,i){var e=this._events&&this._events[t];if(e&&e.length)return!0;if(i)for(var n in this._eventParents)if(this._eventParents[n].listens(t,i))return!0;return!1},once:function(t,i,n){if("object"==typeof t){for(var o in t)this.once(o,t[o],i);return this}var s=e(function(){this.off(t,i,n).off(t,s,n)},this);return this.on(t,i,n).on(t,s,n)},addEventParent:function(t){return this._eventParents=this._eventParents||{},this._eventParents[n(t)]=t,this},removeEventParent:function(t){return this._eventParents&&delete this._eventParents[n(t)],this},_propagateEvent:function(t){for(var e in this._eventParents)this._eventParents[e].fire(t.type,i({layer:t.target,propagatedFrom:t.target},t),!0)}};hi.addEventListener=hi.on,hi.removeEventListener=hi.clearAllEventListeners=hi.off,hi.addOneTimeEventListener=hi.once,hi.fireEvent=hi.fire,hi.hasEventListeners=hi.listens;var ui=v.extend(hi),li=Math.trunc||function(t){return t>0?Math.floor(t):Math.ceil(t)};x.prototype={clone:function(){return new x(this.x,this.y)},add:function(t){return this.clone()._add(w(t))},_add:function(t){return this.x+=t.x,this.y+=t.y,this},subtract:function(t){return this.clone()._subtract(w(t))},_subtract:function(t){return this.x-=t.x,this.y-=t.y,this},divideBy:function(t){return this.clone()._divideBy(t)},_divideBy:function(t){return this.x/=t,this.y/=t,this},multiplyBy:function(t){return this.clone()._multiplyBy(t)},_multiplyBy:function(t){return this.x*=t,this.y*=t,this},scaleBy:function(t){return new x(this.x*t.x,this.y*t.y)},unscaleBy:function(t){return new x(this.x/t.x,this.y/t.y)},round:function(){return this.clone()._round()},_round:function(){return this.x=Math.round(this.x),this.y=Math.round(this.y),this},floor:function(){return this.clone()._floor()},_floor:function(){return this.x=Math.floor(this.x),this.y=Math.floor(this.y),this},ceil:function(){return this.clone()._ceil()},_ceil:function(){return this.x=Math.ceil(this.x),this.y=Math.ceil(this.y),this},trunc:function(){return this.clone()._trunc()},_trunc:function(){return this.x=li(this.x),this.y=li(this.y),this},distanceTo:function(t){var i=(t=w(t)).x-this.x,e=t.y-this.y;return Math.sqrt(i*i+e*e)},equals:function(t){return(t=w(t)).x===this.x&&t.y===this.y},contains:function(t){return t=w(t),Math.abs(t.x)<=Math.abs(this.x)&&Math.abs(t.y)<=Math.abs(this.y)},toString:function(){return"Point("+a(this.x)+", "+a(this.y)+")"}},P.prototype={extend:function(t){return t=w(t),this.min||this.max?(this.min.x=Math.min(t.x,this.min.x),this.max.x=Math.max(t.x,this.max.x),this.min.y=Math.min(t.y,this.min.y),this.max.y=Math.max(t.y,this.max.y)):(this.min=t.clone(),this.max=t.clone()),this},getCenter:function(t){return new x((this.min.x+this.max.x)/2,(this.min.y+this.max.y)/2,t)},getBottomLeft:function(){return new x(this.min.x,this.max.y)},getTopRight:function(){return new x(this.max.x,this.min.y)},getTopLeft:function(){return this.min},getBottomRight:function(){return this.max},getSize:function(){return this.max.subtract(this.min)},contains:function(t){var i,e;return(t="number"==typeof t[0]||t instanceof x?w(t):b(t))instanceof P?(i=t.min,e=t.max):i=e=t,i.x>=this.min.x&&e.x<=this.max.x&&i.y>=this.min.y&&e.y<=this.max.y},intersects:function(t){t=b(t);var i=this.min,e=this.max,n=t.min,o=t.max,s=o.x>=i.x&&n.x<=e.x,r=o.y>=i.y&&n.y<=e.y;return s&&r},overlaps:function(t){t=b(t);var i=this.min,e=this.max,n=t.min,o=t.max,s=o.x>i.x&&n.x<e.x,r=o.y>i.y&&n.y<e.y;return s&&r},isValid:function(){return!(!this.min||!this.max)}},T.prototype={extend:function(t){var i,e,n=this._southWest,o=this._northEast;if(t instanceof M)i=t,e=t;else{if(!(t instanceof T))return t?this.extend(C(t)||z(t)):this;if(i=t._southWest,e=t._northEast,!i||!e)return this}return n||o?(n.lat=Math.min(i.lat,n.lat),n.lng=Math.min(i.lng,n.lng),o.lat=Math.max(e.lat,o.lat),o.lng=Math.max(e.lng,o.lng)):(this._southWest=new M(i.lat,i.lng),this._northEast=new M(e.lat,e.lng)),this},pad:function(t){var i=this._southWest,e=this._northEast,n=Math.abs(i.lat-e.lat)*t,o=Math.abs(i.lng-e.lng)*t;return new T(new M(i.lat-n,i.lng-o),new M(e.lat+n,e.lng+o))},getCenter:function(){return new M((this._southWest.lat+this._northEast.lat)/2,(this._southWest.lng+this._northEast.lng)/2)},getSouthWest:function(){return this._southWest},getNorthEast:function(){return this._northEast},getNorthWest:function(){return new M(this.getNorth(),this.getWest())},getSouthEast:function(){return new M(this.getSouth(),this.getEast())},getWest:function(){return this._southWest.lng},getSouth:function(){return this._southWest.lat},getEast:function(){return this._northEast.lng},getNorth:function(){return this._northEast.lat},contains:function(t){t="number"==typeof t[0]||t instanceof M||"lat"in t?C(t):z(t);var i,e,n=this._southWest,o=this._northEast;return t instanceof T?(i=t.getSouthWest(),e=t.getNorthEast()):i=e=t,i.lat>=n.lat&&e.lat<=o.lat&&i.lng>=n.lng&&e.lng<=o.lng},intersects:function(t){t=z(t);var i=this._southWest,e=this._northEast,n=t.getSouthWest(),o=t.getNorthEast(),s=o.lat>=i.lat&&n.lat<=e.lat,r=o.lng>=i.lng&&n.lng<=e.lng;return s&&r},overlaps:function(t){t=z(t);var i=this._southWest,e=this._northEast,n=t.getSouthWest(),o=t.getNorthEast(),s=o.lat>i.lat&&n.lat<e.lat,r=o.lng>i.lng&&n.lng<e.lng;return s&&r},toBBoxString:function(){return[this.getWest(),this.getSouth(),this.getEast(),this.getNorth()].join(",")},equals:function(t,i){return!!t&&(t=z(t),this._southWest.equals(t.getSouthWest(),i)&&this._northEast.equals(t.getNorthEast(),i))},isValid:function(){return!(!this._southWest||!this._northEast)}},M.prototype={equals:function(t,i){return!!t&&(t=C(t),Math.max(Math.abs(this.lat-t.lat),Math.abs(this.lng-t.lng))<=(void 0===i?1e-9:i))},toString:function(t){return"LatLng("+a(this.lat,t)+", "+a(this.lng,t)+")"},distanceTo:function(t){return _i.distance(this,C(t))},wrap:function(){return _i.wrapLatLng(this)},toBounds:function(t){var i=180*t/40075017,e=i/Math.cos(Math.PI/180*this.lat);return z([this.lat-i,this.lng-e],[this.lat+i,this.lng+e])},clone:function(){return new M(this.lat,this.lng,this.alt)}};var ci={latLngToPoint:function(t,i){var e=this.projection.project(t),n=this.scale(i);return this.transformation._transform(e,n)},pointToLatLng:function(t,i){var e=this.scale(i),n=this.transformation.untransform(t,e);return this.projection.unproject(n)},project:function(t){return this.projection.project(t)},unproject:function(t){return this.projection.unproject(t)},scale:function(t){return 256*Math.pow(2,t)},zoom:function(t){return Math.log(t/256)/Math.LN2},getProjectedBounds:function(t){if(this.infinite)return null;var i=this.projection.bounds,e=this.scale(t);return new P(this.transformation.transform(i.min,e),this.transformation.transform(i.max,e))},infinite:!1,wrapLatLng:function(t){var i=this.wrapLng?s(t.lng,this.wrapLng,!0):t.lng;return new M(this.wrapLat?s(t.lat,this.wrapLat,!0):t.lat,i,t.alt)},wrapLatLngBounds:function(t){var i=t.getCenter(),e=this.wrapLatLng(i),n=i.lat-e.lat,o=i.lng-e.lng;if(0===n&&0===o)return t;var s=t.getSouthWest(),r=t.getNorthEast();return new T(new M(s.lat-n,s.lng-o),new M(r.lat-n,r.lng-o))}},_i=i({},ci,{wrapLng:[-180,180],R:6371e3,distance:function(t,i){var e=Math.PI/180,n=t.lat*e,o=i.lat*e,s=Math.sin((i.lat-t.lat)*e/2),r=Math.sin((i.lng-t.lng)*e/2),a=s*s+Math.cos(n)*Math.cos(o)*r*r,h=2*Math.atan2(Math.sqrt(a),Math.sqrt(1-a));return this.R*h}}),di={R:6378137,MAX_LATITUDE:85.0511287798,project:function(t){var i=Math.PI/180,e=this.MAX_LATITUDE,n=Math.max(Math.min(e,t.lat),-e),o=Math.sin(n*i);return new x(this.R*t.lng*i,this.R*Math.log((1+o)/(1-o))/2)},unproject:function(t){var i=180/Math.PI;return new M((2*Math.atan(Math.exp(t.y/this.R))-Math.PI/2)*i,t.x*i/this.R)},bounds:function(){var t=6378137*Math.PI;return new P([-t,-t],[t,t])}()};Z.prototype={transform:function(t,i){return this._transform(t.clone(),i)},_transform:function(t,i){return i=i||1,t.x=i*(this._a*t.x+this._b),t.y=i*(this._c*t.y+this._d),t},untransform:function(t,i){return i=i||1,new x((t.x/i-this._b)/this._a,(t.y/i-this._d)/this._c)}};var pi,mi,fi,gi,vi=i({},_i,{code:"EPSG:3857",projection:di,transformation:function(){var t=.5/(Math.PI*di.R);return S(t,.5,-t,.5)}()}),yi=i({},vi,{code:"EPSG:900913"}),xi=document.documentElement.style,wi="ActiveXObject"in window,Li=wi&&!document.addEventListener,Pi="msLaunchUri"in navigator&&!("documentMode"in document),bi=A("webkit"),Ti=A("android"),zi=A("android 2")||A("android 3"),Mi=parseInt(/WebKit\/([0-9]+)|$/.exec(navigator.userAgent)[1],10),Ci=Ti&&A("Google")&&Mi<537&&!("AudioNode"in window),Zi=!!window.opera,Si=A("chrome"),Ei=A("gecko")&&!bi&&!Zi&&!wi,ki=!Si&&A("safari"),Ai=A("phantom"),Ii="OTransition"in xi,Bi=0===navigator.platform.indexOf("Win"),Oi=wi&&"transition"in xi,Ri="WebKitCSSMatrix"in window&&"m11"in new window.WebKitCSSMatrix&&!zi,Di="MozPerspective"in xi,Ni=!window.L_DISABLE_3D&&(Oi||Ri||Di)&&!Ii&&!Ai,ji="undefined"!=typeof orientation||A("mobile"),Wi=ji&&bi,Hi=ji&&Ri,Fi=!window.PointerEvent&&window.MSPointerEvent,Ui=!(!window.PointerEvent&&!Fi),Vi=!window.L_NO_TOUCH&&(Ui||"ontouchstart"in window||window.DocumentTouch&&document instanceof window.DocumentTouch),qi=ji&&Zi,Gi=ji&&Ei,Ki=(window.devicePixelRatio||window.screen.deviceXDPI/window.screen.logicalXDPI)>1,Yi=!!document.createElement("canvas").getContext,Xi=!(!document.createElementNS||!E("svg").createSVGRect),Ji=!Xi&&function(){try{var t=document.createElement("div");t.innerHTML='<v:shape adj="1"/>';var i=t.firstChild;return i.style.behavior="url(#default#VML)",i&&"object"==typeof i.adj}catch(t){return!1}}(),$i=(Object.freeze||Object)({ie:wi,ielt9:Li,edge:Pi,webkit:bi,android:Ti,android23:zi,androidStock:Ci,opera:Zi,chrome:Si,gecko:Ei,safari:ki,phantom:Ai,opera12:Ii,win:Bi,ie3d:Oi,webkit3d:Ri,gecko3d:Di,any3d:Ni,mobile:ji,mobileWebkit:Wi,mobileWebkit3d:Hi,msPointer:Fi,pointer:Ui,touch:Vi,mobileOpera:qi,mobileGecko:Gi,retina:Ki,canvas:Yi,svg:Xi,vml:Ji}),Qi=Fi?"MSPointerDown":"pointerdown",te=Fi?"MSPointerMove":"pointermove",ie=Fi?"MSPointerUp":"pointerup",ee=Fi?"MSPointerCancel":"pointercancel",ne=["INPUT","SELECT","OPTION"],oe={},se=!1,re=0,ae=Fi?"MSPointerDown":Ui?"pointerdown":"touchstart",he=Fi?"MSPointerUp":Ui?"pointerup":"touchend",ue="_leaflet_",le="_leaflet_events",ce=Bi&&Si?2*window.devicePixelRatio:Ei?window.devicePixelRatio:1,_e={},de=(Object.freeze||Object)({on:V,off:q,stopPropagation:Y,disableScrollPropagation:X,disableClickPropagation:J,preventDefault:$,stop:Q,getMousePosition:tt,getWheelDelta:it,fakeStop:et,skipped:nt,isExternalTarget:ot,addListener:V,removeListener:q}),pe=xt(["transform","WebkitTransform","OTransform","MozTransform","msTransform"]),me=xt(["webkitTransition","transition","OTransition","MozTransition","msTransition"]),fe="webkitTransition"===me||"OTransition"===me?me+"End":"transitionend";if("onselectstart"in document)mi=function(){V(window,"selectstart",$)},fi=function(){q(window,"selectstart",$)};else{var ge=xt(["userSelect","WebkitUserSelect","OUserSelect","MozUserSelect","msUserSelect"]);mi=function(){if(ge){var t=document.documentElement.style;gi=t[ge],t[ge]="none"}},fi=function(){ge&&(document.documentElement.style[ge]=gi,gi=void 0)}}var ve,ye,xe=(Object.freeze||Object)({TRANSFORM:pe,TRANSITION:me,TRANSITION_END:fe,get:rt,getStyle:at,create:ht,remove:ut,empty:lt,toFront:ct,toBack:_t,hasClass:dt,addClass:pt,removeClass:mt,setClass:ft,getClass:gt,setOpacity:vt,testProp:xt,setTransform:wt,setPosition:Lt,getPosition:Pt,disableTextSelection:mi,enableTextSelection:fi,disableImageDrag:bt,enableImageDrag:Tt,preventOutline:zt,restoreOutline:Mt}),we=ui.extend({run:function(t,i,e,n){this.stop(),this._el=t,this._inProgress=!0,this._duration=e||.25,this._easeOutPower=1/Math.max(n||.5,.2),this._startPos=Pt(t),this._offset=i.subtract(this._startPos),this._startTime=+new Date,this.fire("start"),this._animate()},stop:function(){this._inProgress&&(this._step(!0),this._complete())},_animate:function(){this._animId=f(this._animate,this),this._step()},_step:function(t){var i=+new Date-this._startTime,e=1e3*this._duration;i<e?this._runFrame(this._easeOut(i/e),t):(this._runFrame(1),this._complete())},_runFrame:function(t,i){var e=this._startPos.add(this._offset.multiplyBy(t));i&&e._round(),Lt(this._el,e),this.fire("step")},_complete:function(){g(this._animId),this._inProgress=!1,this.fire("end")},_easeOut:function(t){return 1-Math.pow(1-t,this._easeOutPower)}}),Le=ui.extend({options:{crs:vi,center:void 0,zoom:void 0,minZoom:void 0,maxZoom:void 0,layers:[],maxBounds:void 0,renderer:void 0,zoomAnimation:!0,zoomAnimationThreshold:4,fadeAnimation:!0,markerZoomAnimation:!0,transform3DLimit:8388608,zoomSnap:1,zoomDelta:1,trackResize:!0},initialize:function(t,i){i=l(this,i),this._initContainer(t),this._initLayout(),this._onResize=e(this._onResize,this),this._initEvents(),i.maxBounds&&this.setMaxBounds(i.maxBounds),void 0!==i.zoom&&(this._zoom=this._limitZoom(i.zoom)),i.center&&void 0!==i.zoom&&this.setView(C(i.center),i.zoom,{reset:!0}),this._handlers=[],this._layers={},this._zoomBoundLayers={},this._sizeChanged=!0,this.callInitHooks(),this._zoomAnimated=me&&Ni&&!qi&&this.options.zoomAnimation,this._zoomAnimated&&(this._createAnimProxy(),V(this._proxy,fe,this._catchTransitionEnd,this)),this._addLayers(this.options.layers)},setView:function(t,e,n){return e=void 0===e?this._zoom:this._limitZoom(e),t=this._limitCenter(C(t),e,this.options.maxBounds),n=n||{},this._stop(),this._loaded&&!n.reset&&!0!==n&&(void 0!==n.animate&&(n.zoom=i({animate:n.animate},n.zoom),n.pan=i({animate:n.animate,duration:n.duration},n.pan)),this._zoom!==e?this._tryAnimatedZoom&&this._tryAnimatedZoom(t,e,n.zoom):this._tryAnimatedPan(t,n.pan))?(clearTimeout(this._sizeTimer),this):(this._resetView(t,e),this)},setZoom:function(t,i){return this._loaded?this.setView(this.getCenter(),t,{zoom:i}):(this._zoom=t,this)},zoomIn:function(t,i){return t=t||(Ni?this.options.zoomDelta:1),this.setZoom(this._zoom+t,i)},zoomOut:function(t,i){return t=t||(Ni?this.options.zoomDelta:1),this.setZoom(this._zoom-t,i)},setZoomAround:function(t,i,e){var n=this.getZoomScale(i),o=this.getSize().divideBy(2),s=(t instanceof x?t:this.latLngToContainerPoint(t)).subtract(o).multiplyBy(1-1/n),r=this.containerPointToLatLng(o.add(s));return this.setView(r,i,{zoom:e})},_getBoundsCenterZoom:function(t,i){i=i||{},t=t.getBounds?t.getBounds():z(t);var e=w(i.paddingTopLeft||i.padding||[0,0]),n=w(i.paddingBottomRight||i.padding||[0,0]),o=this.getBoundsZoom(t,!1,e.add(n));if((o="number"==typeof i.maxZoom?Math.min(i.maxZoom,o):o)===1/0)return{center:t.getCenter(),zoom:o};var s=n.subtract(e).divideBy(2),r=this.project(t.getSouthWest(),o),a=this.project(t.getNorthEast(),o);return{center:this.unproject(r.add(a).divideBy(2).add(s),o),zoom:o}},fitBounds:function(t,i){if(!(t=z(t)).isValid())throw new Error("Bounds are not valid.");var e=this._getBoundsCenterZoom(t,i);return this.setView(e.center,e.zoom,i)},fitWorld:function(t){return this.fitBounds([[-90,-180],[90,180]],t)},panTo:function(t,i){return this.setView(t,this._zoom,{pan:i})},panBy:function(t,i){if(t=w(t).round(),i=i||{},!t.x&&!t.y)return this.fire("moveend");if(!0!==i.animate&&!this.getSize().contains(t))return this._resetView(this.unproject(this.project(this.getCenter()).add(t)),this.getZoom()),this;if(this._panAnim||(this._panAnim=new we,this._panAnim.on({step:this._onPanTransitionStep,end:this._onPanTransitionEnd},this)),i.noMoveStart||this.fire("movestart"),!1!==i.animate){pt(this._mapPane,"leaflet-pan-anim");var e=this._getMapPanePos().subtract(t).round();this._panAnim.run(this._mapPane,e,i.duration||.25,i.easeLinearity)}else this._rawPanBy(t),this.fire("move").fire("moveend");return this},flyTo:function(t,i,e){function n(t){var i=(g*g-m*m+(t?-1:1)*x*x*v*v)/(2*(t?g:m)*x*v),e=Math.sqrt(i*i+1)-i;return e<1e-9?-18:Math.log(e)}function o(t){return(Math.exp(t)-Math.exp(-t))/2}function s(t){return(Math.exp(t)+Math.exp(-t))/2}function r(t){return o(t)/s(t)}function a(t){return m*(s(w)/s(w+y*t))}function h(t){return m*(s(w)*r(w+y*t)-o(w))/x}function u(t){return 1-Math.pow(1-t,1.5)}function l(){var e=(Date.now()-L)/b,n=u(e)*P;e<=1?(this._flyToFrame=f(l,this),this._move(this.unproject(c.add(_.subtract(c).multiplyBy(h(n)/v)),p),this.getScaleZoom(m/a(n),p),{flyTo:!0})):this._move(t,i)._moveEnd(!0)}if(!1===(e=e||{}).animate||!Ni)return this.setView(t,i,e);this._stop();var c=this.project(this.getCenter()),_=this.project(t),d=this.getSize(),p=this._zoom;t=C(t),i=void 0===i?p:i;var m=Math.max(d.x,d.y),g=m*this.getZoomScale(p,i),v=_.distanceTo(c)||1,y=1.42,x=y*y,w=n(0),L=Date.now(),P=(n(1)-w)/y,b=e.duration?1e3*e.duration:1e3*P*.8;return this._moveStart(!0,e.noMoveStart),l.call(this),this},flyToBounds:function(t,i){var e=this._getBoundsCenterZoom(t,i);return this.flyTo(e.center,e.zoom,i)},setMaxBounds:function(t){return(t=z(t)).isValid()?(this.options.maxBounds&&this.off("moveend",this._panInsideMaxBounds),this.options.maxBounds=t,this._loaded&&this._panInsideMaxBounds(),this.on("moveend",this._panInsideMaxBounds)):(this.options.maxBounds=null,this.off("moveend",this._panInsideMaxBounds))},setMinZoom:function(t){var i=this.options.minZoom;return this.options.minZoom=t,this._loaded&&i!==t&&(this.fire("zoomlevelschange"),this.getZoom()<this.options.minZoom)?this.setZoom(t):this},setMaxZoom:function(t){var i=this.options.maxZoom;return this.options.maxZoom=t,this._loaded&&i!==t&&(this.fire("zoomlevelschange"),this.getZoom()>this.options.maxZoom)?this.setZoom(t):this},panInsideBounds:function(t,i){this._enforcingBounds=!0;var e=this.getCenter(),n=this._limitCenter(e,this._zoom,z(t));return e.equals(n)||this.panTo(n,i),this._enforcingBounds=!1,this},invalidateSize:function(t){if(!this._loaded)return this;t=i({animate:!1,pan:!0},!0===t?{animate:!0}:t);var n=this.getSize();this._sizeChanged=!0,this._lastCenter=null;var o=this.getSize(),s=n.divideBy(2).round(),r=o.divideBy(2).round(),a=s.subtract(r);return a.x||a.y?(t.animate&&t.pan?this.panBy(a):(t.pan&&this._rawPanBy(a),this.fire("move"),t.debounceMoveend?(clearTimeout(this._sizeTimer),this._sizeTimer=setTimeout(e(this.fire,this,"moveend"),200)):this.fire("moveend")),this.fire("resize",{oldSize:n,newSize:o})):this},stop:function(){return this.setZoom(this._limitZoom(this._zoom)),this.options.zoomSnap||this.fire("viewreset"),this._stop()},locate:function(t){if(t=this._locateOptions=i({timeout:1e4,watch:!1},t),!("geolocation"in navigator))return this._handleGeolocationError({code:0,message:"Geolocation not supported."}),this;var n=e(this._handleGeolocationResponse,this),o=e(this._handleGeolocationError,this);return t.watch?this._locationWatchId=navigator.geolocation.watchPosition(n,o,t):navigator.geolocation.getCurrentPosition(n,o,t),this},stopLocate:function(){return navigator.geolocation&&navigator.geolocation.clearWatch&&navigator.geolocation.clearWatch(this._locationWatchId),this._locateOptions&&(this._locateOptions.setView=!1),this},_handleGeolocationError:function(t){var i=t.code,e=t.message||(1===i?"permission denied":2===i?"position unavailable":"timeout");this._locateOptions.setView&&!this._loaded&&this.fitWorld(),this.fire("locationerror",{code:i,message:"Geolocation error: "+e+"."})},_handleGeolocationResponse:function(t){var i=new M(t.coords.latitude,t.coords.longitude),e=i.toBounds(t.coords.accuracy),n=this._locateOptions;if(n.setView){var o=this.getBoundsZoom(e);this.setView(i,n.maxZoom?Math.min(o,n.maxZoom):o)}var s={latlng:i,bounds:e,timestamp:t.timestamp};for(var r in t.coords)"number"==typeof t.coords[r]&&(s[r]=t.coords[r]);this.fire("locationfound",s)},addHandler:function(t,i){if(!i)return this;var e=this[t]=new i(this);return this._handlers.push(e),this.options[t]&&e.enable(),this},remove:function(){if(this._initEvents(!0),this._containerId!==this._container._leaflet_id)throw new Error("Map container is being reused by another instance");try{delete this._container._leaflet_id,delete this._containerId}catch(t){this._container._leaflet_id=void 0,this._containerId=void 0}void 0!==this._locationWatchId&&this.stopLocate(),this._stop(),ut(this._mapPane),this._clearControlPos&&this._clearControlPos(),this._clearHandlers(),this._loaded&&this.fire("unload");var t;for(t in this._layers)this._layers[t].remove();for(t in this._panes)ut(this._panes[t]);return this._layers=[],this._panes=[],delete this._mapPane,delete this._renderer,this},createPane:function(t,i){var e=ht("div","leaflet-pane"+(t?" leaflet-"+t.replace("Pane","")+"-pane":""),i||this._mapPane);return t&&(this._panes[t]=e),e},getCenter:function(){return this._checkIfLoaded(),this._lastCenter&&!this._moved()?this._lastCenter:this.layerPointToLatLng(this._getCenterLayerPoint())},getZoom:function(){return this._zoom},getBounds:function(){var t=this.getPixelBounds();return new T(this.unproject(t.getBottomLeft()),this.unproject(t.getTopRight()))},getMinZoom:function(){return void 0===this.options.minZoom?this._layersMinZoom||0:this.options.minZoom},getMaxZoom:function(){return void 0===this.options.maxZoom?void 0===this._layersMaxZoom?1/0:this._layersMaxZoom:this.options.maxZoom},getBoundsZoom:function(t,i,e){t=z(t),e=w(e||[0,0]);var n=this.getZoom()||0,o=this.getMinZoom(),s=this.getMaxZoom(),r=t.getNorthWest(),a=t.getSouthEast(),h=this.getSize().subtract(e),u=b(this.project(a,n),this.project(r,n)).getSize(),l=Ni?this.options.zoomSnap:1,c=h.x/u.x,_=h.y/u.y,d=i?Math.max(c,_):Math.min(c,_);return n=this.getScaleZoom(d,n),l&&(n=Math.round(n/(l/100))*(l/100),n=i?Math.ceil(n/l)*l:Math.floor(n/l)*l),Math.max(o,Math.min(s,n))},getSize:function(){return this._size&&!this._sizeChanged||(this._size=new x(this._container.clientWidth||0,this._container.clientHeight||0),this._sizeChanged=!1),this._size.clone()},getPixelBounds:function(t,i){var e=this._getTopLeftPoint(t,i);return new P(e,e.add(this.getSize()))},getPixelOrigin:function(){return this._checkIfLoaded(),this._pixelOrigin},getPixelWorldBounds:function(t){return this.options.crs.getProjectedBounds(void 0===t?this.getZoom():t)},getPane:function(t){return"string"==typeof t?this._panes[t]:t},getPanes:function(){return this._panes},getContainer:function(){return this._container},getZoomScale:function(t,i){var e=this.options.crs;return i=void 0===i?this._zoom:i,e.scale(t)/e.scale(i)},getScaleZoom:function(t,i){var e=this.options.crs;i=void 0===i?this._zoom:i;var n=e.zoom(t*e.scale(i));return isNaN(n)?1/0:n},project:function(t,i){return i=void 0===i?this._zoom:i,this.options.crs.latLngToPoint(C(t),i)},unproject:function(t,i){return i=void 0===i?this._zoom:i,this.options.crs.pointToLatLng(w(t),i)},layerPointToLatLng:function(t){var i=w(t).add(this.getPixelOrigin());return this.unproject(i)},latLngToLayerPoint:function(t){return this.project(C(t))._round()._subtract(this.getPixelOrigin())},wrapLatLng:function(t){return this.options.crs.wrapLatLng(C(t))},wrapLatLngBounds:function(t){return this.options.crs.wrapLatLngBounds(z(t))},distance:function(t,i){return this.options.crs.distance(C(t),C(i))},containerPointToLayerPoint:function(t){return w(t).subtract(this._getMapPanePos())},layerPointToContainerPoint:function(t){return w(t).add(this._getMapPanePos())},containerPointToLatLng:function(t){var i=this.containerPointToLayerPoint(w(t));return this.layerPointToLatLng(i)},latLngToContainerPoint:function(t){return this.layerPointToContainerPoint(this.latLngToLayerPoint(C(t)))},mouseEventToContainerPoint:function(t){return tt(t,this._container)},mouseEventToLayerPoint:function(t){return this.containerPointToLayerPoint(this.mouseEventToContainerPoint(t))},mouseEventToLatLng:function(t){return this.layerPointToLatLng(this.mouseEventToLayerPoint(t))},_initContainer:function(t){var i=this._container=rt(t);if(!i)throw new Error("Map container not found.");if(i._leaflet_id)throw new Error("Map container is already initialized.");V(i,"scroll",this._onScroll,this),this._containerId=n(i)},_initLayout:function(){var t=this._container;this._fadeAnimated=this.options.fadeAnimation&&Ni,pt(t,"leaflet-container"+(Vi?" leaflet-touch":"")+(Ki?" leaflet-retina":"")+(Li?" leaflet-oldie":"")+(ki?" leaflet-safari":"")+(this._fadeAnimated?" leaflet-fade-anim":""));var i=at(t,"position");"absolute"!==i&&"relative"!==i&&"fixed"!==i&&(t.style.position="relative"),this._initPanes(),this._initControlPos&&this._initControlPos()},_initPanes:function(){var t=this._panes={};this._paneRenderers={},this._mapPane=this.createPane("mapPane",this._container),Lt(this._mapPane,new x(0,0)),this.createPane("tilePane"),this.createPane("shadowPane"),this.createPane("overlayPane"),this.createPane("markerPane"),this.createPane("tooltipPane"),this.createPane("popupPane"),this.options.markerZoomAnimation||(pt(t.markerPane,"leaflet-zoom-hide"),pt(t.shadowPane,"leaflet-zoom-hide"))},_resetView:function(t,i){Lt(this._mapPane,new x(0,0));var e=!this._loaded;this._loaded=!0,i=this._limitZoom(i),this.fire("viewprereset");var n=this._zoom!==i;this._moveStart(n,!1)._move(t,i)._moveEnd(n),this.fire("viewreset"),e&&this.fire("load")},_moveStart:function(t,i){return t&&this.fire("zoomstart"),i||this.fire("movestart"),this},_move:function(t,i,e){void 0===i&&(i=this._zoom);var n=this._zoom!==i;return this._zoom=i,this._lastCenter=t,this._pixelOrigin=this._getNewPixelOrigin(t),(n||e&&e.pinch)&&this.fire("zoom",e),this.fire("move",e)},_moveEnd:function(t){return t&&this.fire("zoomend"),this.fire("moveend")},_stop:function(){return g(this._flyToFrame),this._panAnim&&this._panAnim.stop(),this},_rawPanBy:function(t){Lt(this._mapPane,this._getMapPanePos().subtract(t))},_getZoomSpan:function(){return this.getMaxZoom()-this.getMinZoom()},_panInsideMaxBounds:function(){this._enforcingBounds||this.panInsideBounds(this.options.maxBounds)},_checkIfLoaded:function(){if(!this._loaded)throw new Error("Set map center and zoom first.")},_initEvents:function(t){this._targets={},this._targets[n(this._container)]=this;var i=t?q:V;i(this._container,"click dblclick mousedown mouseup mouseover mouseout mousemove contextmenu keypress",this._handleDOMEvent,this),this.options.trackResize&&i(window,"resize",this._onResize,this),Ni&&this.options.transform3DLimit&&(t?this.off:this.on).call(this,"moveend",this._onMoveEnd)},_onResize:function(){g(this._resizeRequest),this._resizeRequest=f(function(){this.invalidateSize({debounceMoveend:!0})},this)},_onScroll:function(){this._container.scrollTop=0,this._container.scrollLeft=0},_onMoveEnd:function(){var t=this._getMapPanePos();Math.max(Math.abs(t.x),Math.abs(t.y))>=this.options.transform3DLimit&&this._resetView(this.getCenter(),this.getZoom())},_findEventTargets:function(t,i){for(var e,o=[],s="mouseout"===i||"mouseover"===i,r=t.target||t.srcElement,a=!1;r;){if((e=this._targets[n(r)])&&("click"===i||"preclick"===i)&&!t._simulated&&this._draggableMoved(e)){a=!0;break}if(e&&e.listens(i,!0)){if(s&&!ot(r,t))break;if(o.push(e),s)break}if(r===this._container)break;r=r.parentNode}return o.length||a||s||!ot(r,t)||(o=[this]),o},_handleDOMEvent:function(t){if(this._loaded&&!nt(t)){var i=t.type;"mousedown"!==i&&"keypress"!==i||zt(t.target||t.srcElement),this._fireDOMEvent(t,i)}},_mouseEvents:["click","dblclick","mouseover","mouseout","contextmenu"],_fireDOMEvent:function(t,e,n){if("click"===t.type){var o=i({},t);o.type="preclick",this._fireDOMEvent(o,o.type,n)}if(!t._stopped&&(n=(n||[]).concat(this._findEventTargets(t,e))).length){var s=n[0];"contextmenu"===e&&s.listens(e,!0)&&$(t);var r={originalEvent:t};if("keypress"!==t.type){var a=s.getLatLng&&(!s._radius||s._radius<=10);r.containerPoint=a?this.latLngToContainerPoint(s.getLatLng()):this.mouseEventToContainerPoint(t),r.layerPoint=this.containerPointToLayerPoint(r.containerPoint),r.latlng=a?s.getLatLng():this.layerPointToLatLng(r.layerPoint)}for(var h=0;h<n.length;h++)if(n[h].fire(e,r,!0),r.originalEvent._stopped||!1===n[h].options.bubblingMouseEvents&&-1!==d(this._mouseEvents,e))return}},_draggableMoved:function(t){return(t=t.dragging&&t.dragging.enabled()?t:this).dragging&&t.dragging.moved()||this.boxZoom&&this.boxZoom.moved()},_clearHandlers:function(){for(var t=0,i=this._handlers.length;t<i;t++)this._handlers[t].disable()},whenReady:function(t,i){return this._loaded?t.call(i||this,{target:this}):this.on("load",t,i),this},_getMapPanePos:function(){return Pt(this._mapPane)||new x(0,0)},_moved:function(){var t=this._getMapPanePos();return t&&!t.equals([0,0])},_getTopLeftPoint:function(t,i){return(t&&void 0!==i?this._getNewPixelOrigin(t,i):this.getPixelOrigin()).subtract(this._getMapPanePos())},_getNewPixelOrigin:function(t,i){var e=this.getSize()._divideBy(2);return this.project(t,i)._subtract(e)._add(this._getMapPanePos())._round()},_latLngToNewLayerPoint:function(t,i,e){var n=this._getNewPixelOrigin(e,i);return this.project(t,i)._subtract(n)},_latLngBoundsToNewLayerBounds:function(t,i,e){var n=this._getNewPixelOrigin(e,i);return b([this.project(t.getSouthWest(),i)._subtract(n),this.project(t.getNorthWest(),i)._subtract(n),this.project(t.getSouthEast(),i)._subtract(n),this.project(t.getNorthEast(),i)._subtract(n)])},_getCenterLayerPoint:function(){return this.containerPointToLayerPoint(this.getSize()._divideBy(2))},_getCenterOffset:function(t){return this.latLngToLayerPoint(t).subtract(this._getCenterLayerPoint())},_limitCenter:function(t,i,e){if(!e)return t;var n=this.project(t,i),o=this.getSize().divideBy(2),s=new P(n.subtract(o),n.add(o)),r=this._getBoundsOffset(s,e,i);return r.round().equals([0,0])?t:this.unproject(n.add(r),i)},_limitOffset:function(t,i){if(!i)return t;var e=this.getPixelBounds(),n=new P(e.min.add(t),e.max.add(t));return t.add(this._getBoundsOffset(n,i))},_getBoundsOffset:function(t,i,e){var n=b(this.project(i.getNorthEast(),e),this.project(i.getSouthWest(),e)),o=n.min.subtract(t.min),s=n.max.subtract(t.max);return new x(this._rebound(o.x,-s.x),this._rebound(o.y,-s.y))},_rebound:function(t,i){return t+i>0?Math.round(t-i)/2:Math.max(0,Math.ceil(t))-Math.max(0,Math.floor(i))},_limitZoom:function(t){var i=this.getMinZoom(),e=this.getMaxZoom(),n=Ni?this.options.zoomSnap:1;return n&&(t=Math.round(t/n)*n),Math.max(i,Math.min(e,t))},_onPanTransitionStep:function(){this.fire("move")},_onPanTransitionEnd:function(){mt(this._mapPane,"leaflet-pan-anim"),this.fire("moveend")},_tryAnimatedPan:function(t,i){var e=this._getCenterOffset(t)._trunc();return!(!0!==(i&&i.animate)&&!this.getSize().contains(e))&&(this.panBy(e,i),!0)},_createAnimProxy:function(){var t=this._proxy=ht("div","leaflet-proxy leaflet-zoom-animated");this._panes.mapPane.appendChild(t),this.on("zoomanim",function(t){var i=pe,e=this._proxy.style[i];wt(this._proxy,this.project(t.center,t.zoom),this.getZoomScale(t.zoom,1)),e===this._proxy.style[i]&&this._animatingZoom&&this._onZoomTransitionEnd()},this),this.on("load moveend",function(){var t=this.getCenter(),i=this.getZoom();wt(this._proxy,this.project(t,i),this.getZoomScale(i,1))},this),this._on("unload",this._destroyAnimProxy,this)},_destroyAnimProxy:function(){ut(this._proxy),delete this._proxy},_catchTransitionEnd:function(t){this._animatingZoom&&t.propertyName.indexOf("transform")>=0&&this._onZoomTransitionEnd()},_nothingToAnimate:function(){return!this._container.getElementsByClassName("leaflet-zoom-animated").length},_tryAnimatedZoom:function(t,i,e){if(this._animatingZoom)return!0;if(e=e||{},!this._zoomAnimated||!1===e.animate||this._nothingToAnimate()||Math.abs(i-this._zoom)>this.options.zoomAnimationThreshold)return!1;var n=this.getZoomScale(i),o=this._getCenterOffset(t)._divideBy(1-1/n);return!(!0!==e.animate&&!this.getSize().contains(o))&&(f(function(){this._moveStart(!0,!1)._animateZoom(t,i,!0)},this),!0)},_animateZoom:function(t,i,n,o){this._mapPane&&(n&&(this._animatingZoom=!0,this._animateToCenter=t,this._animateToZoom=i,pt(this._mapPane,"leaflet-zoom-anim")),this.fire("zoomanim",{center:t,zoom:i,noUpdate:o}),setTimeout(e(this._onZoomTransitionEnd,this),250))},_onZoomTransitionEnd:function(){this._animatingZoom&&(this._mapPane&&mt(this._mapPane,"leaflet-zoom-anim"),this._animatingZoom=!1,this._move(this._animateToCenter,this._animateToZoom),f(function(){this._moveEnd(!0)},this))}}),Pe=v.extend({options:{position:"topright"},initialize:function(t){l(this,t)},getPosition:function(){return this.options.position},setPosition:function(t){var i=this._map;return i&&i.removeControl(this),this.options.position=t,i&&i.addControl(this),this},getContainer:function(){return this._container},addTo:function(t){this.remove(),this._map=t;var i=this._container=this.onAdd(t),e=this.getPosition(),n=t._controlCorners[e];return pt(i,"leaflet-control"),-1!==e.indexOf("bottom")?n.insertBefore(i,n.firstChild):n.appendChild(i),this},remove:function(){return this._map?(ut(this._container),this.onRemove&&this.onRemove(this._map),this._map=null,this):this},_refocusOnMap:function(t){this._map&&t&&t.screenX>0&&t.screenY>0&&this._map.getContainer().focus()}}),be=function(t){return new Pe(t)};Le.include({addControl:function(t){return t.addTo(this),this},removeControl:function(t){return t.remove(),this},_initControlPos:function(){function t(t,o){var s=e+t+" "+e+o;i[t+o]=ht("div",s,n)}var i=this._controlCorners={},e="leaflet-",n=this._controlContainer=ht("div",e+"control-container",this._container);t("top","left"),t("top","right"),t("bottom","left"),t("bottom","right")},_clearControlPos:function(){for(var t in this._controlCorners)ut(this._controlCorners[t]);ut(this._controlContainer),delete this._controlCorners,delete this._controlContainer}});var Te=Pe.extend({options:{collapsed:!0,position:"topright",autoZIndex:!0,hideSingleBase:!1,sortLayers:!1,sortFunction:function(t,i,e,n){return e<n?-1:n<e?1:0}},initialize:function(t,i,e){l(this,e),this._layerControlInputs=[],this._layers=[],this._lastZIndex=0,this._handlingClick=!1;for(var n in t)this._addLayer(t[n],n);for(n in i)this._addLayer(i[n],n,!0)},onAdd:function(t){this._initLayout(),this._update(),this._map=t,t.on("zoomend",this._checkDisabledLayers,this);for(var i=0;i<this._layers.length;i++)this._layers[i].layer.on("add remove",this._onLayerChange,this);return this._container},addTo:function(t){return Pe.prototype.addTo.call(this,t),this._expandIfNotCollapsed()},onRemove:function(){this._map.off("zoomend",this._checkDisabledLayers,this);for(var t=0;t<this._layers.length;t++)this._layers[t].layer.off("add remove",this._onLayerChange,this)},addBaseLayer:function(t,i){return this._addLayer(t,i),this._map?this._update():this},addOverlay:function(t,i){return this._addLayer(t,i,!0),this._map?this._update():this},removeLayer:function(t){t.off("add remove",this._onLayerChange,this);var i=this._getLayer(n(t));return i&&this._layers.splice(this._layers.indexOf(i),1),this._map?this._update():this},expand:function(){pt(this._container,"leaflet-control-layers-expanded"),this._form.style.height=null;var t=this._map.getSize().y-(this._container.offsetTop+50);return t<this._form.clientHeight?(pt(this._form,"leaflet-control-layers-scrollbar"),this._form.style.height=t+"px"):mt(this._form,"leaflet-control-layers-scrollbar"),this._checkDisabledLayers(),this},collapse:function(){return mt(this._container,"leaflet-control-layers-expanded"),this},_initLayout:function(){var t="leaflet-control-layers",i=this._container=ht("div",t),e=this.options.collapsed;i.setAttribute("aria-haspopup",!0),J(i),X(i);var n=this._form=ht("form",t+"-list");e&&(this._map.on("click",this.collapse,this),Ti||V(i,{mouseenter:this.expand,mouseleave:this.collapse},this));var o=this._layersLink=ht("a",t+"-toggle",i);o.href="#",o.title="Layers",Vi?(V(o,"click",Q),V(o,"click",this.expand,this)):V(o,"focus",this.expand,this),e||this.expand(),this._baseLayersList=ht("div",t+"-base",n),this._separator=ht("div",t+"-separator",n),this._overlaysList=ht("div",t+"-overlays",n),i.appendChild(n)},_getLayer:function(t){for(var i=0;i<this._layers.length;i++)if(this._layers[i]&&n(this._layers[i].layer)===t)return this._layers[i]},_addLayer:function(t,i,n){this._map&&t.on("add remove",this._onLayerChange,this),this._layers.push({layer:t,name:i,overlay:n}),this.options.sortLayers&&this._layers.sort(e(function(t,i){return this.options.sortFunction(t.layer,i.layer,t.name,i.name)},this)),this.options.autoZIndex&&t.setZIndex&&(this._lastZIndex++,t.setZIndex(this._lastZIndex)),this._expandIfNotCollapsed()},_update:function(){if(!this._container)return this;lt(this._baseLayersList),lt(this._overlaysList),this._layerControlInputs=[];var t,i,e,n,o=0;for(e=0;e<this._layers.length;e++)n=this._layers[e],this._addItem(n),i=i||n.overlay,t=t||!n.overlay,o+=n.overlay?0:1;return this.options.hideSingleBase&&(t=t&&o>1,this._baseLayersList.style.display=t?"":"none"),this._separator.style.display=i&&t?"":"none",this},_onLayerChange:function(t){this._handlingClick||this._update();var i=this._getLayer(n(t.target)),e=i.overlay?"add"===t.type?"overlayadd":"overlayremove":"add"===t.type?"baselayerchange":null;e&&this._map.fire(e,i)},_createRadioElement:function(t,i){var e='<input type="radio" class="leaflet-control-layers-selector" name="'+t+'"'+(i?' checked="checked"':"")+"/>",n=document.createElement("div");return n.innerHTML=e,n.firstChild},_addItem:function(t){var i,e=document.createElement("label"),o=this._map.hasLayer(t.layer);t.overlay?((i=document.createElement("input")).type="checkbox",i.className="leaflet-control-layers-selector",i.defaultChecked=o):i=this._createRadioElement("leaflet-base-layers",o),this._layerControlInputs.push(i),i.layerId=n(t.layer),V(i,"click",this._onInputClick,this);var s=document.createElement("span");s.innerHTML=" "+t.name;var r=document.createElement("div");return e.appendChild(r),r.appendChild(i),r.appendChild(s),(t.overlay?this._overlaysList:this._baseLayersList).appendChild(e),this._checkDisabledLayers(),e},_onInputClick:function(){var t,i,e=this._layerControlInputs,n=[],o=[];this._handlingClick=!0;for(var s=e.length-1;s>=0;s--)t=e[s],i=this._getLayer(t.layerId).layer,t.checked?n.push(i):t.checked||o.push(i);for(s=0;s<o.length;s++)this._map.hasLayer(o[s])&&this._map.removeLayer(o[s]);for(s=0;s<n.length;s++)this._map.hasLayer(n[s])||this._map.addLayer(n[s]);this._handlingClick=!1,this._refocusOnMap()},_checkDisabledLayers:function(){for(var t,i,e=this._layerControlInputs,n=this._map.getZoom(),o=e.length-1;o>=0;o--)t=e[o],i=this._getLayer(t.layerId).layer,t.disabled=void 0!==i.options.minZoom&&n<i.options.minZoom||void 0!==i.options.maxZoom&&n>i.options.maxZoom},_expandIfNotCollapsed:function(){return this._map&&!this.options.collapsed&&this.expand(),this},_expand:function(){return this.expand()},_collapse:function(){return this.collapse()}}),ze=Pe.extend({options:{position:"topleft",zoomInText:"+",zoomInTitle:"Zoom in",zoomOutText:"&#x2212;",zoomOutTitle:"Zoom out"},onAdd:function(t){var i="leaflet-control-zoom",e=ht("div",i+" leaflet-bar"),n=this.options;return this._zoomInButton=this._createButton(n.zoomInText,n.zoomInTitle,i+"-in",e,this._zoomIn),this._zoomOutButton=this._createButton(n.zoomOutText,n.zoomOutTitle,i+"-out",e,this._zoomOut),this._updateDisabled(),t.on("zoomend zoomlevelschange",this._updateDisabled,this),e},onRemove:function(t){t.off("zoomend zoomlevelschange",this._updateDisabled,this)},disable:function(){return this._disabled=!0,this._updateDisabled(),this},enable:function(){return this._disabled=!1,this._updateDisabled(),this},_zoomIn:function(t){!this._disabled&&this._map._zoom<this._map.getMaxZoom()&&this._map.zoomIn(this._map.options.zoomDelta*(t.shiftKey?3:1))},_zoomOut:function(t){!this._disabled&&this._map._zoom>this._map.getMinZoom()&&this._map.zoomOut(this._map.options.zoomDelta*(t.shiftKey?3:1))},_createButton:function(t,i,e,n,o){var s=ht("a",e,n);return s.innerHTML=t,s.href="#",s.title=i,s.setAttribute("role","button"),s.setAttribute("aria-label",i),J(s),V(s,"click",Q),V(s,"click",o,this),V(s,"click",this._refocusOnMap,this),s},_updateDisabled:function(){var t=this._map,i="leaflet-disabled";mt(this._zoomInButton,i),mt(this._zoomOutButton,i),(this._disabled||t._zoom===t.getMinZoom())&&pt(this._zoomOutButton,i),(this._disabled||t._zoom===t.getMaxZoom())&&pt(this._zoomInButton,i)}});Le.mergeOptions({zoomControl:!0}),Le.addInitHook(function(){this.options.zoomControl&&(this.zoomControl=new ze,this.addControl(this.zoomControl))});var Me=Pe.extend({options:{position:"bottomleft",maxWidth:100,metric:!0,imperial:!0},onAdd:function(t){var i=ht("div","leaflet-control-scale"),e=this.options;return this._addScales(e,"leaflet-control-scale-line",i),t.on(e.updateWhenIdle?"moveend":"move",this._update,this),t.whenReady(this._update,this),i},onRemove:function(t){t.off(this.options.updateWhenIdle?"moveend":"move",this._update,this)},_addScales:function(t,i,e){t.metric&&(this._mScale=ht("div",i,e)),t.imperial&&(this._iScale=ht("div",i,e))},_update:function(){var t=this._map,i=t.getSize().y/2,e=t.distance(t.containerPointToLatLng([0,i]),t.containerPointToLatLng([this.options.maxWidth,i]));this._updateScales(e)},_updateScales:function(t){this.options.metric&&t&&this._updateMetric(t),this.options.imperial&&t&&this._updateImperial(t)},_updateMetric:function(t){var i=this._getRoundNum(t),e=i<1e3?i+" m":i/1e3+" km";this._updateScale(this._mScale,e,i/t)},_updateImperial:function(t){var i,e,n,o=3.2808399*t;o>5280?(i=o/5280,e=this._getRoundNum(i),this._updateScale(this._iScale,e+" mi",e/i)):(n=this._getRoundNum(o),this._updateScale(this._iScale,n+" ft",n/o))},_updateScale:function(t,i,e){t.style.width=Math.round(this.options.maxWidth*e)+"px",t.innerHTML=i},_getRoundNum:function(t){var i=Math.pow(10,(Math.floor(t)+"").length-1),e=t/i;return e=e>=10?10:e>=5?5:e>=3?3:e>=2?2:1,i*e}}),Ce=Pe.extend({options:{position:"bottomright",prefix:'<a href="https://leafletjs.com" title="A JS library for interactive maps">Leaflet</a>'},initialize:function(t){l(this,t),this._attributions={}},onAdd:function(t){t.attributionControl=this,this._container=ht("div","leaflet-control-attribution"),J(this._container);for(var i in t._layers)t._layers[i].getAttribution&&this.addAttribution(t._layers[i].getAttribution());return this._update(),this._container},setPrefix:function(t){return this.options.prefix=t,this._update(),this},addAttribution:function(t){return t?(this._attributions[t]||(this._attributions[t]=0),this._attributions[t]++,this._update(),this):this},removeAttribution:function(t){return t?(this._attributions[t]&&(this._attributions[t]--,this._update()),this):this},_update:function(){if(this._map){var t=[];for(var i in this._attributions)this._attributions[i]&&t.push(i);var e=[];this.options.prefix&&e.push(this.options.prefix),t.length&&e.push(t.join(", ")),this._container.innerHTML=e.join(" | ")}}});Le.mergeOptions({attributionControl:!0}),Le.addInitHook(function(){this.options.attributionControl&&(new Ce).addTo(this)});Pe.Layers=Te,Pe.Zoom=ze,Pe.Scale=Me,Pe.Attribution=Ce,be.layers=function(t,i,e){return new Te(t,i,e)},be.zoom=function(t){return new ze(t)},be.scale=function(t){return new Me(t)},be.attribution=function(t){return new Ce(t)};var Ze=v.extend({initialize:function(t){this._map=t},enable:function(){return this._enabled?this:(this._enabled=!0,this.addHooks(),this)},disable:function(){return this._enabled?(this._enabled=!1,this.removeHooks(),this):this},enabled:function(){return!!this._enabled}});Ze.addTo=function(t,i){return t.addHandler(i,this),this};var Se,Ee={Events:hi},ke=Vi?"touchstart mousedown":"mousedown",Ae={mousedown:"mouseup",touchstart:"touchend",pointerdown:"touchend",MSPointerDown:"touchend"},Ie={mousedown:"mousemove",touchstart:"touchmove",pointerdown:"touchmove",MSPointerDown:"touchmove"},Be=ui.extend({options:{clickTolerance:3},initialize:function(t,i,e,n){l(this,n),this._element=t,this._dragStartTarget=i||t,this._preventOutline=e},enable:function(){this._enabled||(V(this._dragStartTarget,ke,this._onDown,this),this._enabled=!0)},disable:function(){this._enabled&&(Be._dragging===this&&this.finishDrag(),q(this._dragStartTarget,ke,this._onDown,this),this._enabled=!1,this._moved=!1)},_onDown:function(t){if(!t._simulated&&this._enabled&&(this._moved=!1,!dt(this._element,"leaflet-zoom-anim")&&!(Be._dragging||t.shiftKey||1!==t.which&&1!==t.button&&!t.touches||(Be._dragging=this,this._preventOutline&&zt(this._element),bt(),mi(),this._moving)))){this.fire("down");var i=t.touches?t.touches[0]:t;this._startPoint=new x(i.clientX,i.clientY),V(document,Ie[t.type],this._onMove,this),V(document,Ae[t.type],this._onUp,this)}},_onMove:function(t){if(!t._simulated&&this._enabled)if(t.touches&&t.touches.length>1)this._moved=!0;else{var i=t.touches&&1===t.touches.length?t.touches[0]:t,e=new x(i.clientX,i.clientY).subtract(this._startPoint);(e.x||e.y)&&(Math.abs(e.x)+Math.abs(e.y)<this.options.clickTolerance||($(t),this._moved||(this.fire("dragstart"),this._moved=!0,this._startPos=Pt(this._element).subtract(e),pt(document.body,"leaflet-dragging"),this._lastTarget=t.target||t.srcElement,window.SVGElementInstance&&this._lastTarget instanceof SVGElementInstance&&(this._lastTarget=this._lastTarget.correspondingUseElement),pt(this._lastTarget,"leaflet-drag-target")),this._newPos=this._startPos.add(e),this._moving=!0,g(this._animRequest),this._lastEvent=t,this._animRequest=f(this._updatePosition,this,!0)))}},_updatePosition:function(){var t={originalEvent:this._lastEvent};this.fire("predrag",t),Lt(this._element,this._newPos),this.fire("drag",t)},_onUp:function(t){!t._simulated&&this._enabled&&this.finishDrag()},finishDrag:function(){mt(document.body,"leaflet-dragging"),this._lastTarget&&(mt(this._lastTarget,"leaflet-drag-target"),this._lastTarget=null);for(var t in Ie)q(document,Ie[t],this._onMove,this),q(document,Ae[t],this._onUp,this);Tt(),fi(),this._moved&&this._moving&&(g(this._animRequest),this.fire("dragend",{distance:this._newPos.distanceTo(this._startPos)})),this._moving=!1,Be._dragging=!1}}),Oe=(Object.freeze||Object)({simplify:Ct,pointToSegmentDistance:Zt,closestPointOnSegment:function(t,i,e){return Rt(t,i,e)},clipSegment:At,_getEdgeIntersection:It,_getBitCode:Bt,_sqClosestPointOnSegment:Rt,isFlat:Dt,_flat:Nt}),Re=(Object.freeze||Object)({clipPolygon:jt}),De={project:function(t){return new x(t.lng,t.lat)},unproject:function(t){return new M(t.y,t.x)},bounds:new P([-180,-90],[180,90])},Ne={R:6378137,R_MINOR:6356752.314245179,bounds:new P([-20037508.34279,-15496570.73972],[20037508.34279,18764656.23138]),project:function(t){var i=Math.PI/180,e=this.R,n=t.lat*i,o=this.R_MINOR/e,s=Math.sqrt(1-o*o),r=s*Math.sin(n),a=Math.tan(Math.PI/4-n/2)/Math.pow((1-r)/(1+r),s/2);return n=-e*Math.log(Math.max(a,1e-10)),new x(t.lng*i*e,n)},unproject:function(t){for(var i,e=180/Math.PI,n=this.R,o=this.R_MINOR/n,s=Math.sqrt(1-o*o),r=Math.exp(-t.y/n),a=Math.PI/2-2*Math.atan(r),h=0,u=.1;h<15&&Math.abs(u)>1e-7;h++)i=s*Math.sin(a),i=Math.pow((1-i)/(1+i),s/2),a+=u=Math.PI/2-2*Math.atan(r*i)-a;return new M(a*e,t.x*e/n)}},je=(Object.freeze||Object)({LonLat:De,Mercator:Ne,SphericalMercator:di}),We=i({},_i,{code:"EPSG:3395",projection:Ne,transformation:function(){var t=.5/(Math.PI*Ne.R);return S(t,.5,-t,.5)}()}),He=i({},_i,{code:"EPSG:4326",projection:De,transformation:S(1/180,1,-1/180,.5)}),Fe=i({},ci,{projection:De,transformation:S(1,0,-1,0),scale:function(t){return Math.pow(2,t)},zoom:function(t){return Math.log(t)/Math.LN2},distance:function(t,i){var e=i.lng-t.lng,n=i.lat-t.lat;return Math.sqrt(e*e+n*n)},infinite:!0});ci.Earth=_i,ci.EPSG3395=We,ci.EPSG3857=vi,ci.EPSG900913=yi,ci.EPSG4326=He,ci.Simple=Fe;var Ue=ui.extend({options:{pane:"overlayPane",attribution:null,bubblingMouseEvents:!0},addTo:function(t){return t.addLayer(this),this},remove:function(){return this.removeFrom(this._map||this._mapToAdd)},removeFrom:function(t){return t&&t.removeLayer(this),this},getPane:function(t){return this._map.getPane(t?this.options[t]||t:this.options.pane)},addInteractiveTarget:function(t){return this._map._targets[n(t)]=this,this},removeInteractiveTarget:function(t){return delete this._map._targets[n(t)],this},getAttribution:function(){return this.options.attribution},_layerAdd:function(t){var i=t.target;if(i.hasLayer(this)){if(this._map=i,this._zoomAnimated=i._zoomAnimated,this.getEvents){var e=this.getEvents();i.on(e,this),this.once("remove",function(){i.off(e,this)},this)}this.onAdd(i),this.getAttribution&&i.attributionControl&&i.attributionControl.addAttribution(this.getAttribution()),this.fire("add"),i.fire("layeradd",{layer:this})}}});Le.include({addLayer:function(t){if(!t._layerAdd)throw new Error("The provided object is not a Layer.");var i=n(t);return this._layers[i]?this:(this._layers[i]=t,t._mapToAdd=this,t.beforeAdd&&t.beforeAdd(this),this.whenReady(t._layerAdd,t),this)},removeLayer:function(t){var i=n(t);return this._layers[i]?(this._loaded&&t.onRemove(this),t.getAttribution&&this.attributionControl&&this.attributionControl.removeAttribution(t.getAttribution()),delete this._layers[i],this._loaded&&(this.fire("layerremove",{layer:t}),t.fire("remove")),t._map=t._mapToAdd=null,this):this},hasLayer:function(t){return!!t&&n(t)in this._layers},eachLayer:function(t,i){for(var e in this._layers)t.call(i,this._layers[e]);return this},_addLayers:function(t){for(var i=0,e=(t=t?ei(t)?t:[t]:[]).length;i<e;i++)this.addLayer(t[i])},_addZoomLimit:function(t){!isNaN(t.options.maxZoom)&&isNaN(t.options.minZoom)||(this._zoomBoundLayers[n(t)]=t,this._updateZoomLevels())},_removeZoomLimit:function(t){var i=n(t);this._zoomBoundLayers[i]&&(delete this._zoomBoundLayers[i],this._updateZoomLevels())},_updateZoomLevels:function(){var t=1/0,i=-1/0,e=this._getZoomSpan();for(var n in this._zoomBoundLayers){var o=this._zoomBoundLayers[n].options;t=void 0===o.minZoom?t:Math.min(t,o.minZoom),i=void 0===o.maxZoom?i:Math.max(i,o.maxZoom)}this._layersMaxZoom=i===-1/0?void 0:i,this._layersMinZoom=t===1/0?void 0:t,e!==this._getZoomSpan()&&this.fire("zoomlevelschange"),void 0===this.options.maxZoom&&this._layersMaxZoom&&this.getZoom()>this._layersMaxZoom&&this.setZoom(this._layersMaxZoom),void 0===this.options.minZoom&&this._layersMinZoom&&this.getZoom()<this._layersMinZoom&&this.setZoom(this._layersMinZoom)}});var Ve=Ue.extend({initialize:function(t,i){l(this,i),this._layers={};var e,n;if(t)for(e=0,n=t.length;e<n;e++)this.addLayer(t[e])},addLayer:function(t){var i=this.getLayerId(t);return this._layers[i]=t,this._map&&this._map.addLayer(t),this},removeLayer:function(t){var i=t in this._layers?t:this.getLayerId(t);return this._map&&this._layers[i]&&this._map.removeLayer(this._layers[i]),delete this._layers[i],this},hasLayer:function(t){return!!t&&(t in this._layers||this.getLayerId(t)in this._layers)},clearLayers:function(){return this.eachLayer(this.removeLayer,this)},invoke:function(t){var i,e,n=Array.prototype.slice.call(arguments,1);for(i in this._layers)(e=this._layers[i])[t]&&e[t].apply(e,n);return this},onAdd:function(t){this.eachLayer(t.addLayer,t)},onRemove:function(t){this.eachLayer(t.removeLayer,t)},eachLayer:function(t,i){for(var e in this._layers)t.call(i,this._layers[e]);return this},getLayer:function(t){return this._layers[t]},getLayers:function(){var t=[];return this.eachLayer(t.push,t),t},setZIndex:function(t){return this.invoke("setZIndex",t)},getLayerId:function(t){return n(t)}}),qe=Ve.extend({addLayer:function(t){return this.hasLayer(t)?this:(t.addEventParent(this),Ve.prototype.addLayer.call(this,t),this.fire("layeradd",{layer:t}))},removeLayer:function(t){return this.hasLayer(t)?(t in this._layers&&(t=this._layers[t]),t.removeEventParent(this),Ve.prototype.removeLayer.call(this,t),this.fire("layerremove",{layer:t})):this},setStyle:function(t){return this.invoke("setStyle",t)},bringToFront:function(){return this.invoke("bringToFront")},bringToBack:function(){return this.invoke("bringToBack")},getBounds:function(){var t=new T;for(var i in this._layers){var e=this._layers[i];t.extend(e.getBounds?e.getBounds():e.getLatLng())}return t}}),Ge=v.extend({options:{popupAnchor:[0,0],tooltipAnchor:[0,0]},initialize:function(t){l(this,t)},createIcon:function(t){return this._createIcon("icon",t)},createShadow:function(t){return this._createIcon("shadow",t)},_createIcon:function(t,i){var e=this._getIconUrl(t);if(!e){if("icon"===t)throw new Error("iconUrl not set in Icon options (see the docs).");return null}var n=this._createImg(e,i&&"IMG"===i.tagName?i:null);return this._setIconStyles(n,t),n},_setIconStyles:function(t,i){var e=this.options,n=e[i+"Size"];"number"==typeof n&&(n=[n,n]);var o=w(n),s=w("shadow"===i&&e.shadowAnchor||e.iconAnchor||o&&o.divideBy(2,!0));t.className="leaflet-marker-"+i+" "+(e.className||""),s&&(t.style.marginLeft=-s.x+"px",t.style.marginTop=-s.y+"px"),o&&(t.style.width=o.x+"px",t.style.height=o.y+"px")},_createImg:function(t,i){return i=i||document.createElement("img"),i.src=t,i},_getIconUrl:function(t){return Ki&&this.options[t+"RetinaUrl"]||this.options[t+"Url"]}}),Ke=Ge.extend({options:{iconUrl:"marker-icon.png",iconRetinaUrl:"marker-icon-2x.png",shadowUrl:"marker-shadow.png",iconSize:[25,41],iconAnchor:[12,41],popupAnchor:[1,-34],tooltipAnchor:[16,-28],shadowSize:[41,41]},_getIconUrl:function(t){return Ke.imagePath||(Ke.imagePath=this._detectIconPath()),(this.options.imagePath||Ke.imagePath)+Ge.prototype._getIconUrl.call(this,t)},_detectIconPath:function(){var t=ht("div","leaflet-default-icon-path",document.body),i=at(t,"background-image")||at(t,"backgroundImage");return document.body.removeChild(t),i=null===i||0!==i.indexOf("url")?"":i.replace(/^url\(["']?/,"").replace(/marker-icon\.png["']?\)$/,"")}}),Ye=Ze.extend({initialize:function(t){this._marker=t},addHooks:function(){var t=this._marker._icon;this._draggable||(this._draggable=new Be(t,t,!0)),this._draggable.on({dragstart:this._onDragStart,predrag:this._onPreDrag,drag:this._onDrag,dragend:this._onDragEnd},this).enable(),pt(t,"leaflet-marker-draggable")},removeHooks:function(){this._draggable.off({dragstart:this._onDragStart,predrag:this._onPreDrag,drag:this._onDrag,dragend:this._onDragEnd},this).disable(),this._marker._icon&&mt(this._marker._icon,"leaflet-marker-draggable")},moved:function(){return this._draggable&&this._draggable._moved},_adjustPan:function(t){var i=this._marker,e=i._map,n=this._marker.options.autoPanSpeed,o=this._marker.options.autoPanPadding,s=L.DomUtil.getPosition(i._icon),r=e.getPixelBounds(),a=e.getPixelOrigin(),h=b(r.min._subtract(a).add(o),r.max._subtract(a).subtract(o));if(!h.contains(s)){var u=w((Math.max(h.max.x,s.x)-h.max.x)/(r.max.x-h.max.x)-(Math.min(h.min.x,s.x)-h.min.x)/(r.min.x-h.min.x),(Math.max(h.max.y,s.y)-h.max.y)/(r.max.y-h.max.y)-(Math.min(h.min.y,s.y)-h.min.y)/(r.min.y-h.min.y)).multiplyBy(n);e.panBy(u,{animate:!1}),this._draggable._newPos._add(u),this._draggable._startPos._add(u),L.DomUtil.setPosition(i._icon,this._draggable._newPos),this._onDrag(t),this._panRequest=f(this._adjustPan.bind(this,t))}},_onDragStart:function(){this._oldLatLng=this._marker.getLatLng(),this._marker.closePopup().fire("movestart").fire("dragstart")},_onPreDrag:function(t){this._marker.options.autoPan&&(g(this._panRequest),this._panRequest=f(this._adjustPan.bind(this,t)))},_onDrag:function(t){var i=this._marker,e=i._shadow,n=Pt(i._icon),o=i._map.layerPointToLatLng(n);e&&Lt(e,n),i._latlng=o,t.latlng=o,t.oldLatLng=this._oldLatLng,i.fire("move",t).fire("drag",t)},_onDragEnd:function(t){g(this._panRequest),delete this._oldLatLng,this._marker.fire("moveend").fire("dragend",t)}}),Xe=Ue.extend({options:{icon:new Ke,interactive:!0,draggable:!1,autoPan:!1,autoPanPadding:[50,50],autoPanSpeed:10,keyboard:!0,title:"",alt:"",zIndexOffset:0,opacity:1,riseOnHover:!1,riseOffset:250,pane:"markerPane",bubblingMouseEvents:!1},initialize:function(t,i){l(this,i),this._latlng=C(t)},onAdd:function(t){this._zoomAnimated=this._zoomAnimated&&t.options.markerZoomAnimation,this._zoomAnimated&&t.on("zoomanim",this._animateZoom,this),this._initIcon(),this.update()},onRemove:function(t){this.dragging&&this.dragging.enabled()&&(this.options.draggable=!0,this.dragging.removeHooks()),delete this.dragging,this._zoomAnimated&&t.off("zoomanim",this._animateZoom,this),this._removeIcon(),this._removeShadow()},getEvents:function(){return{zoom:this.update,viewreset:this.update}},getLatLng:function(){return this._latlng},setLatLng:function(t){var i=this._latlng;return this._latlng=C(t),this.update(),this.fire("move",{oldLatLng:i,latlng:this._latlng})},setZIndexOffset:function(t){return this.options.zIndexOffset=t,this.update()},setIcon:function(t){return this.options.icon=t,this._map&&(this._initIcon(),this.update()),this._popup&&this.bindPopup(this._popup,this._popup.options),this},getElement:function(){return this._icon},update:function(){if(this._icon&&this._map){var t=this._map.latLngToLayerPoint(this._latlng).round();this._setPos(t)}return this},_initIcon:function(){var t=this.options,i="leaflet-zoom-"+(this._zoomAnimated?"animated":"hide"),e=t.icon.createIcon(this._icon),n=!1;e!==this._icon&&(this._icon&&this._removeIcon(),n=!0,t.title&&(e.title=t.title),"IMG"===e.tagName&&(e.alt=t.alt||"")),pt(e,i),t.keyboard&&(e.tabIndex="0"),this._icon=e,t.riseOnHover&&this.on({mouseover:this._bringToFront,mouseout:this._resetZIndex});var o=t.icon.createShadow(this._shadow),s=!1;o!==this._shadow&&(this._removeShadow(),s=!0),o&&(pt(o,i),o.alt=""),this._shadow=o,t.opacity<1&&this._updateOpacity(),n&&this.getPane().appendChild(this._icon),this._initInteraction(),o&&s&&this.getPane("shadowPane").appendChild(this._shadow)},_removeIcon:function(){this.options.riseOnHover&&this.off({mouseover:this._bringToFront,mouseout:this._resetZIndex}),ut(this._icon),this.removeInteractiveTarget(this._icon),this._icon=null},_removeShadow:function(){this._shadow&&ut(this._shadow),this._shadow=null},_setPos:function(t){Lt(this._icon,t),this._shadow&&Lt(this._shadow,t),this._zIndex=t.y+this.options.zIndexOffset,this._resetZIndex()},_updateZIndex:function(t){this._icon.style.zIndex=this._zIndex+t},_animateZoom:function(t){var i=this._map._latLngToNewLayerPoint(this._latlng,t.zoom,t.center).round();this._setPos(i)},_initInteraction:function(){if(this.options.interactive&&(pt(this._icon,"leaflet-interactive"),this.addInteractiveTarget(this._icon),Ye)){var t=this.options.draggable;this.dragging&&(t=this.dragging.enabled(),this.dragging.disable()),this.dragging=new Ye(this),t&&this.dragging.enable()}},setOpacity:function(t){return this.options.opacity=t,this._map&&this._updateOpacity(),this},_updateOpacity:function(){var t=this.options.opacity;vt(this._icon,t),this._shadow&&vt(this._shadow,t)},_bringToFront:function(){this._updateZIndex(this.options.riseOffset)},_resetZIndex:function(){this._updateZIndex(0)},_getPopupAnchor:function(){return this.options.icon.options.popupAnchor},_getTooltipAnchor:function(){return this.options.icon.options.tooltipAnchor}}),Je=Ue.extend({options:{stroke:!0,color:"#3388ff",weight:3,opacity:1,lineCap:"round",lineJoin:"round",dashArray:null,dashOffset:null,fill:!1,fillColor:null,fillOpacity:.2,fillRule:"evenodd",interactive:!0,bubblingMouseEvents:!0},beforeAdd:function(t){this._renderer=t.getRenderer(this)},onAdd:function(){this._renderer._initPath(this),this._reset(),this._renderer._addPath(this)},onRemove:function(){this._renderer._removePath(this)},redraw:function(){return this._map&&this._renderer._updatePath(this),this},setStyle:function(t){return l(this,t),this._renderer&&this._renderer._updateStyle(this),this},bringToFront:function(){return this._renderer&&this._renderer._bringToFront(this),this},bringToBack:function(){return this._renderer&&this._renderer._bringToBack(this),this},getElement:function(){return this._path},_reset:function(){this._project(),this._update()},_clickTolerance:function(){return(this.options.stroke?this.options.weight/2:0)+this._renderer.options.tolerance}}),$e=Je.extend({options:{fill:!0,radius:10},initialize:function(t,i){l(this,i),this._latlng=C(t),this._radius=this.options.radius},setLatLng:function(t){return this._latlng=C(t),this.redraw(),this.fire("move",{latlng:this._latlng})},getLatLng:function(){return this._latlng},setRadius:function(t){return this.options.radius=this._radius=t,this.redraw()},getRadius:function(){return this._radius},setStyle:function(t){var i=t&&t.radius||this._radius;return Je.prototype.setStyle.call(this,t),this.setRadius(i),this},_project:function(){this._point=this._map.latLngToLayerPoint(this._latlng),this._updateBounds()},_updateBounds:function(){var t=this._radius,i=this._radiusY||t,e=this._clickTolerance(),n=[t+e,i+e];this._pxBounds=new P(this._point.subtract(n),this._point.add(n))},_update:function(){this._map&&this._updatePath()},_updatePath:function(){this._renderer._updateCircle(this)},_empty:function(){return this._radius&&!this._renderer._bounds.intersects(this._pxBounds)},_containsPoint:function(t){return t.distanceTo(this._point)<=this._radius+this._clickTolerance()}}),Qe=$e.extend({initialize:function(t,e,n){if("number"==typeof e&&(e=i({},n,{radius:e})),l(this,e),this._latlng=C(t),isNaN(this.options.radius))throw new Error("Circle radius cannot be NaN");this._mRadius=this.options.radius},setRadius:function(t){return this._mRadius=t,this.redraw()},getRadius:function(){return this._mRadius},getBounds:function(){var t=[this._radius,this._radiusY||this._radius];return new T(this._map.layerPointToLatLng(this._point.subtract(t)),this._map.layerPointToLatLng(this._point.add(t)))},setStyle:Je.prototype.setStyle,_project:function(){var t=this._latlng.lng,i=this._latlng.lat,e=this._map,n=e.options.crs;if(n.distance===_i.distance){var o=Math.PI/180,s=this._mRadius/_i.R/o,r=e.project([i+s,t]),a=e.project([i-s,t]),h=r.add(a).divideBy(2),u=e.unproject(h).lat,l=Math.acos((Math.cos(s*o)-Math.sin(i*o)*Math.sin(u*o))/(Math.cos(i*o)*Math.cos(u*o)))/o;(isNaN(l)||0===l)&&(l=s/Math.cos(Math.PI/180*i)),this._point=h.subtract(e.getPixelOrigin()),this._radius=isNaN(l)?0:h.x-e.project([u,t-l]).x,this._radiusY=h.y-r.y}else{var c=n.unproject(n.project(this._latlng).subtract([this._mRadius,0]));this._point=e.latLngToLayerPoint(this._latlng),this._radius=this._point.x-e.latLngToLayerPoint(c).x}this._updateBounds()}}),tn=Je.extend({options:{smoothFactor:1,noClip:!1},initialize:function(t,i){l(this,i),this._setLatLngs(t)},getLatLngs:function(){return this._latlngs},setLatLngs:function(t){return this._setLatLngs(t),this.redraw()},isEmpty:function(){return!this._latlngs.length},closestLayerPoint:function(t){for(var i,e,n=1/0,o=null,s=Rt,r=0,a=this._parts.length;r<a;r++)for(var h=this._parts[r],u=1,l=h.length;u<l;u++){var c=s(t,i=h[u-1],e=h[u],!0);c<n&&(n=c,o=s(t,i,e))}return o&&(o.distance=Math.sqrt(n)),o},getCenter:function(){if(!this._map)throw new Error("Must add layer to map before using getCenter()");var t,i,e,n,o,s,r,a=this._rings[0],h=a.length;if(!h)return null;for(t=0,i=0;t<h-1;t++)i+=a[t].distanceTo(a[t+1])/2;if(0===i)return this._map.layerPointToLatLng(a[0]);for(t=0,n=0;t<h-1;t++)if(o=a[t],s=a[t+1],e=o.distanceTo(s),(n+=e)>i)return r=(n-i)/e,this._map.layerPointToLatLng([s.x-r*(s.x-o.x),s.y-r*(s.y-o.y)])},getBounds:function(){return this._bounds},addLatLng:function(t,i){return i=i||this._defaultShape(),t=C(t),i.push(t),this._bounds.extend(t),this.redraw()},_setLatLngs:function(t){this._bounds=new T,this._latlngs=this._convertLatLngs(t)},_defaultShape:function(){return Dt(this._latlngs)?this._latlngs:this._latlngs[0]},_convertLatLngs:function(t){for(var i=[],e=Dt(t),n=0,o=t.length;n<o;n++)e?(i[n]=C(t[n]),this._bounds.extend(i[n])):i[n]=this._convertLatLngs(t[n]);return i},_project:function(){var t=new P;this._rings=[],this._projectLatlngs(this._latlngs,this._rings,t);var i=this._clickTolerance(),e=new x(i,i);this._bounds.isValid()&&t.isValid()&&(t.min._subtract(e),t.max._add(e),this._pxBounds=t)},_projectLatlngs:function(t,i,e){var n,o,s=t[0]instanceof M,r=t.length;if(s){for(o=[],n=0;n<r;n++)o[n]=this._map.latLngToLayerPoint(t[n]),e.extend(o[n]);i.push(o)}else for(n=0;n<r;n++)this._projectLatlngs(t[n],i,e)},_clipPoints:function(){var t=this._renderer._bounds;if(this._parts=[],this._pxBounds&&this._pxBounds.intersects(t))if(this.options.noClip)this._parts=this._rings;else{var i,e,n,o,s,r,a,h=this._parts;for(i=0,n=0,o=this._rings.length;i<o;i++)for(e=0,s=(a=this._rings[i]).length;e<s-1;e++)(r=At(a[e],a[e+1],t,e,!0))&&(h[n]=h[n]||[],h[n].push(r[0]),r[1]===a[e+1]&&e!==s-2||(h[n].push(r[1]),n++))}},_simplifyPoints:function(){for(var t=this._parts,i=this.options.smoothFactor,e=0,n=t.length;e<n;e++)t[e]=Ct(t[e],i)},_update:function(){this._map&&(this._clipPoints(),this._simplifyPoints(),this._updatePath())},_updatePath:function(){this._renderer._updatePoly(this)},_containsPoint:function(t,i){var e,n,o,s,r,a,h=this._clickTolerance();if(!this._pxBounds||!this._pxBounds.contains(t))return!1;for(e=0,s=this._parts.length;e<s;e++)for(n=0,o=(r=(a=this._parts[e]).length)-1;n<r;o=n++)if((i||0!==n)&&Zt(t,a[o],a[n])<=h)return!0;return!1}});tn._flat=Nt;var en=tn.extend({options:{fill:!0},isEmpty:function(){return!this._latlngs.length||!this._latlngs[0].length},getCenter:function(){if(!this._map)throw new Error("Must add layer to map before using getCenter()");var t,i,e,n,o,s,r,a,h,u=this._rings[0],l=u.length;if(!l)return null;for(s=r=a=0,t=0,i=l-1;t<l;i=t++)e=u[t],n=u[i],o=e.y*n.x-n.y*e.x,r+=(e.x+n.x)*o,a+=(e.y+n.y)*o,s+=3*o;return h=0===s?u[0]:[r/s,a/s],this._map.layerPointToLatLng(h)},_convertLatLngs:function(t){var i=tn.prototype._convertLatLngs.call(this,t),e=i.length;return e>=2&&i[0]instanceof M&&i[0].equals(i[e-1])&&i.pop(),i},_setLatLngs:function(t){tn.prototype._setLatLngs.call(this,t),Dt(this._latlngs)&&(this._latlngs=[this._latlngs])},_defaultShape:function(){return Dt(this._latlngs[0])?this._latlngs[0]:this._latlngs[0][0]},_clipPoints:function(){var t=this._renderer._bounds,i=this.options.weight,e=new x(i,i);if(t=new P(t.min.subtract(e),t.max.add(e)),this._parts=[],this._pxBounds&&this._pxBounds.intersects(t))if(this.options.noClip)this._parts=this._rings;else for(var n,o=0,s=this._rings.length;o<s;o++)(n=jt(this._rings[o],t,!0)).length&&this._parts.push(n)},_updatePath:function(){this._renderer._updatePoly(this,!0)},_containsPoint:function(t){var i,e,n,o,s,r,a,h,u=!1;if(!this._pxBounds.contains(t))return!1;for(o=0,a=this._parts.length;o<a;o++)for(s=0,r=(h=(i=this._parts[o]).length)-1;s<h;r=s++)e=i[s],n=i[r],e.y>t.y!=n.y>t.y&&t.x<(n.x-e.x)*(t.y-e.y)/(n.y-e.y)+e.x&&(u=!u);return u||tn.prototype._containsPoint.call(this,t,!0)}}),nn=qe.extend({initialize:function(t,i){l(this,i),this._layers={},t&&this.addData(t)},addData:function(t){var i,e,n,o=ei(t)?t:t.features;if(o){for(i=0,e=o.length;i<e;i++)((n=o[i]).geometries||n.geometry||n.features||n.coordinates)&&this.addData(n);return this}var s=this.options;if(s.filter&&!s.filter(t))return this;var r=Wt(t,s);return r?(r.feature=Gt(t),r.defaultOptions=r.options,this.resetStyle(r),s.onEachFeature&&s.onEachFeature(t,r),this.addLayer(r)):this},resetStyle:function(t){return t.options=i({},t.defaultOptions),this._setLayerStyle(t,this.options.style),this},setStyle:function(t){return this.eachLayer(function(i){this._setLayerStyle(i,t)},this)},_setLayerStyle:function(t,i){"function"==typeof i&&(i=i(t.feature)),t.setStyle&&t.setStyle(i)}}),on={toGeoJSON:function(t){return qt(this,{type:"Point",coordinates:Ut(this.getLatLng(),t)})}};Xe.include(on),Qe.include(on),$e.include(on),tn.include({toGeoJSON:function(t){var i=!Dt(this._latlngs),e=Vt(this._latlngs,i?1:0,!1,t);return qt(this,{type:(i?"Multi":"")+"LineString",coordinates:e})}}),en.include({toGeoJSON:function(t){var i=!Dt(this._latlngs),e=i&&!Dt(this._latlngs[0]),n=Vt(this._latlngs,e?2:i?1:0,!0,t);return i||(n=[n]),qt(this,{type:(e?"Multi":"")+"Polygon",coordinates:n})}}),Ve.include({toMultiPoint:function(t){var i=[];return this.eachLayer(function(e){i.push(e.toGeoJSON(t).geometry.coordinates)}),qt(this,{type:"MultiPoint",coordinates:i})},toGeoJSON:function(t){var i=this.feature&&this.feature.geometry&&this.feature.geometry.type;if("MultiPoint"===i)return this.toMultiPoint(t);var e="GeometryCollection"===i,n=[];return this.eachLayer(function(i){if(i.toGeoJSON){var o=i.toGeoJSON(t);if(e)n.push(o.geometry);else{var s=Gt(o);"FeatureCollection"===s.type?n.push.apply(n,s.features):n.push(s)}}}),e?qt(this,{geometries:n,type:"GeometryCollection"}):{type:"FeatureCollection",features:n}}});var sn=Kt,rn=Ue.extend({options:{opacity:1,alt:"",interactive:!1,crossOrigin:!1,errorOverlayUrl:"",zIndex:1,className:""},initialize:function(t,i,e){this._url=t,this._bounds=z(i),l(this,e)},onAdd:function(){this._image||(this._initImage(),this.options.opacity<1&&this._updateOpacity()),this.options.interactive&&(pt(this._image,"leaflet-interactive"),this.addInteractiveTarget(this._image)),this.getPane().appendChild(this._image),this._reset()},onRemove:function(){ut(this._image),this.options.interactive&&this.removeInteractiveTarget(this._image)},setOpacity:function(t){return this.options.opacity=t,this._image&&this._updateOpacity(),this},setStyle:function(t){return t.opacity&&this.setOpacity(t.opacity),this},bringToFront:function(){return this._map&&ct(this._image),this},bringToBack:function(){return this._map&&_t(this._image),this},setUrl:function(t){return this._url=t,this._image&&(this._image.src=t),this},setBounds:function(t){return this._bounds=z(t),this._map&&this._reset(),this},getEvents:function(){var t={zoom:this._reset,viewreset:this._reset};return this._zoomAnimated&&(t.zoomanim=this._animateZoom),t},setZIndex:function(t){return this.options.zIndex=t,this._updateZIndex(),this},getBounds:function(){return this._bounds},getElement:function(){return this._image},_initImage:function(){var t="IMG"===this._url.tagName,i=this._image=t?this._url:ht("img");pt(i,"leaflet-image-layer"),this._zoomAnimated&&pt(i,"leaflet-zoom-animated"),this.options.className&&pt(i,this.options.className),i.onselectstart=r,i.onmousemove=r,i.onload=e(this.fire,this,"load"),i.onerror=e(this._overlayOnError,this,"error"),this.options.crossOrigin&&(i.crossOrigin=""),this.options.zIndex&&this._updateZIndex(),t?this._url=i.src:(i.src=this._url,i.alt=this.options.alt)},_animateZoom:function(t){var i=this._map.getZoomScale(t.zoom),e=this._map._latLngBoundsToNewLayerBounds(this._bounds,t.zoom,t.center).min;wt(this._image,e,i)},_reset:function(){var t=this._image,i=new P(this._map.latLngToLayerPoint(this._bounds.getNorthWest()),this._map.latLngToLayerPoint(this._bounds.getSouthEast())),e=i.getSize();Lt(t,i.min),t.style.width=e.x+"px",t.style.height=e.y+"px"},_updateOpacity:function(){vt(this._image,this.options.opacity)},_updateZIndex:function(){this._image&&void 0!==this.options.zIndex&&null!==this.options.zIndex&&(this._image.style.zIndex=this.options.zIndex)},_overlayOnError:function(){this.fire("error");var t=this.options.errorOverlayUrl;t&&this._url!==t&&(this._url=t,this._image.src=t)}}),an=rn.extend({options:{autoplay:!0,loop:!0},_initImage:function(){var t="VIDEO"===this._url.tagName,i=this._image=t?this._url:ht("video");if(pt(i,"leaflet-image-layer"),this._zoomAnimated&&pt(i,"leaflet-zoom-animated"),i.onselectstart=r,i.onmousemove=r,i.onloadeddata=e(this.fire,this,"load"),t){for(var n=i.getElementsByTagName("source"),o=[],s=0;s<n.length;s++)o.push(n[s].src);this._url=n.length>0?o:[i.src]}else{ei(this._url)||(this._url=[this._url]),i.autoplay=!!this.options.autoplay,i.loop=!!this.options.loop;for(var a=0;a<this._url.length;a++){var h=ht("source");h.src=this._url[a],i.appendChild(h)}}}}),hn=Ue.extend({options:{offset:[0,7],className:"",pane:"popupPane"},initialize:function(t,i){l(this,t),this._source=i},onAdd:function(t){this._zoomAnimated=t._zoomAnimated,this._container||this._initLayout(),t._fadeAnimated&&vt(this._container,0),clearTimeout(this._removeTimeout),this.getPane().appendChild(this._container),this.update(),t._fadeAnimated&&vt(this._container,1),this.bringToFront()},onRemove:function(t){t._fadeAnimated?(vt(this._container,0),this._removeTimeout=setTimeout(e(ut,void 0,this._container),200)):ut(this._container)},getLatLng:function(){return this._latlng},setLatLng:function(t){return this._latlng=C(t),this._map&&(this._updatePosition(),this._adjustPan()),this},getContent:function(){return this._content},setContent:function(t){return this._content=t,this.update(),this},getElement:function(){return this._container},update:function(){this._map&&(this._container.style.visibility="hidden",this._updateContent(),this._updateLayout(),this._updatePosition(),this._container.style.visibility="",this._adjustPan())},getEvents:function(){var t={zoom:this._updatePosition,viewreset:this._updatePosition};return this._zoomAnimated&&(t.zoomanim=this._animateZoom),t},isOpen:function(){return!!this._map&&this._map.hasLayer(this)},bringToFront:function(){return this._map&&ct(this._container),this},bringToBack:function(){return this._map&&_t(this._container),this},_updateContent:function(){if(this._content){var t=this._contentNode,i="function"==typeof this._content?this._content(this._source||this):this._content;if("string"==typeof i)t.innerHTML=i;else{for(;t.hasChildNodes();)t.removeChild(t.firstChild);t.appendChild(i)}this.fire("contentupdate")}},_updatePosition:function(){if(this._map){var t=this._map.latLngToLayerPoint(this._latlng),i=w(this.options.offset),e=this._getAnchor();this._zoomAnimated?Lt(this._container,t.add(e)):i=i.add(t).add(e);var n=this._containerBottom=-i.y,o=this._containerLeft=-Math.round(this._containerWidth/2)+i.x;this._container.style.bottom=n+"px",this._container.style.left=o+"px"}},_getAnchor:function(){return[0,0]}}),un=hn.extend({options:{maxWidth:300,minWidth:50,maxHeight:null,autoPan:!0,autoPanPaddingTopLeft:null,autoPanPaddingBottomRight:null,autoPanPadding:[5,5],keepInView:!1,closeButton:!0,autoClose:!0,closeOnEscapeKey:!0,className:""},openOn:function(t){return t.openPopup(this),this},onAdd:function(t){hn.prototype.onAdd.call(this,t),t.fire("popupopen",{popup:this}),this._source&&(this._source.fire("popupopen",{popup:this},!0),this._source instanceof Je||this._source.on("preclick",Y))},onRemove:function(t){hn.prototype.onRemove.call(this,t),t.fire("popupclose",{popup:this}),this._source&&(this._source.fire("popupclose",{popup:this},!0),this._source instanceof Je||this._source.off("preclick",Y))},getEvents:function(){var t=hn.prototype.getEvents.call(this);return(void 0!==this.options.closeOnClick?this.options.closeOnClick:this._map.options.closePopupOnClick)&&(t.preclick=this._close),this.options.keepInView&&(t.moveend=this._adjustPan),t},_close:function(){this._map&&this._map.closePopup(this)},_initLayout:function(){var t="leaflet-popup",i=this._container=ht("div",t+" "+(this.options.className||"")+" leaflet-zoom-animated"),e=this._wrapper=ht("div",t+"-content-wrapper",i);if(this._contentNode=ht("div",t+"-content",e),J(e),X(this._contentNode),V(e,"contextmenu",Y),this._tipContainer=ht("div",t+"-tip-container",i),this._tip=ht("div",t+"-tip",this._tipContainer),this.options.closeButton){var n=this._closeButton=ht("a",t+"-close-button",i);n.href="#close",n.innerHTML="&#215;",V(n,"click",this._onCloseButtonClick,this)}},_updateLayout:function(){var t=this._contentNode,i=t.style;i.width="",i.whiteSpace="nowrap";var e=t.offsetWidth;e=Math.min(e,this.options.maxWidth),e=Math.max(e,this.options.minWidth),i.width=e+1+"px",i.whiteSpace="",i.height="";var n=t.offsetHeight,o=this.options.maxHeight;o&&n>o?(i.height=o+"px",pt(t,"leaflet-popup-scrolled")):mt(t,"leaflet-popup-scrolled"),this._containerWidth=this._container.offsetWidth},_animateZoom:function(t){var i=this._map._latLngToNewLayerPoint(this._latlng,t.zoom,t.center),e=this._getAnchor();Lt(this._container,i.add(e))},_adjustPan:function(){if(!(!this.options.autoPan||this._map._panAnim&&this._map._panAnim._inProgress)){var t=this._map,i=parseInt(at(this._container,"marginBottom"),10)||0,e=this._container.offsetHeight+i,n=this._containerWidth,o=new x(this._containerLeft,-e-this._containerBottom);o._add(Pt(this._container));var s=t.layerPointToContainerPoint(o),r=w(this.options.autoPanPadding),a=w(this.options.autoPanPaddingTopLeft||r),h=w(this.options.autoPanPaddingBottomRight||r),u=t.getSize(),l=0,c=0;s.x+n+h.x>u.x&&(l=s.x+n-u.x+h.x),s.x-l-a.x<0&&(l=s.x-a.x),s.y+e+h.y>u.y&&(c=s.y+e-u.y+h.y),s.y-c-a.y<0&&(c=s.y-a.y),(l||c)&&t.fire("autopanstart").panBy([l,c])}},_onCloseButtonClick:function(t){this._close(),Q(t)},_getAnchor:function(){return w(this._source&&this._source._getPopupAnchor?this._source._getPopupAnchor():[0,0])}});Le.mergeOptions({closePopupOnClick:!0}),Le.include({openPopup:function(t,i,e){return t instanceof un||(t=new un(e).setContent(t)),i&&t.setLatLng(i),this.hasLayer(t)?this:(this._popup&&this._popup.options.autoClose&&this.closePopup(),this._popup=t,this.addLayer(t))},closePopup:function(t){return t&&t!==this._popup||(t=this._popup,this._popup=null),t&&this.removeLayer(t),this}}),Ue.include({bindPopup:function(t,i){return t instanceof un?(l(t,i),this._popup=t,t._source=this):(this._popup&&!i||(this._popup=new un(i,this)),this._popup.setContent(t)),this._popupHandlersAdded||(this.on({click:this._openPopup,keypress:this._onKeyPress,remove:this.closePopup,move:this._movePopup}),this._popupHandlersAdded=!0),this},unbindPopup:function(){return this._popup&&(this.off({click:this._openPopup,keypress:this._onKeyPress,remove:this.closePopup,move:this._movePopup}),this._popupHandlersAdded=!1,this._popup=null),this},openPopup:function(t,i){if(t instanceof Ue||(i=t,t=this),t instanceof qe)for(var e in this._layers){t=this._layers[e];break}return i||(i=t.getCenter?t.getCenter():t.getLatLng()),this._popup&&this._map&&(this._popup._source=t,this._popup.update(),this._map.openPopup(this._popup,i)),this},closePopup:function(){return this._popup&&this._popup._close(),this},togglePopup:function(t){return this._popup&&(this._popup._map?this.closePopup():this.openPopup(t)),this},isPopupOpen:function(){return!!this._popup&&this._popup.isOpen()},setPopupContent:function(t){return this._popup&&this._popup.setContent(t),this},getPopup:function(){return this._popup},_openPopup:function(t){var i=t.layer||t.target;this._popup&&this._map&&(Q(t),i instanceof Je?this.openPopup(t.layer||t.target,t.latlng):this._map.hasLayer(this._popup)&&this._popup._source===i?this.closePopup():this.openPopup(i,t.latlng))},_movePopup:function(t){this._popup.setLatLng(t.latlng)},_onKeyPress:function(t){13===t.originalEvent.keyCode&&this._openPopup(t)}});var ln=hn.extend({options:{pane:"tooltipPane",offset:[0,0],direction:"auto",permanent:!1,sticky:!1,interactive:!1,opacity:.9},onAdd:function(t){hn.prototype.onAdd.call(this,t),this.setOpacity(this.options.opacity),t.fire("tooltipopen",{tooltip:this}),this._source&&this._source.fire("tooltipopen",{tooltip:this},!0)},onRemove:function(t){hn.prototype.onRemove.call(this,t),t.fire("tooltipclose",{tooltip:this}),this._source&&this._source.fire("tooltipclose",{tooltip:this},!0)},getEvents:function(){var t=hn.prototype.getEvents.call(this);return Vi&&!this.options.permanent&&(t.preclick=this._close),t},_close:function(){this._map&&this._map.closeTooltip(this)},_initLayout:function(){var t="leaflet-tooltip "+(this.options.className||"")+" leaflet-zoom-"+(this._zoomAnimated?"animated":"hide");this._contentNode=this._container=ht("div",t)},_updateLayout:function(){},_adjustPan:function(){},_setPosition:function(t){var i=this._map,e=this._container,n=i.latLngToContainerPoint(i.getCenter()),o=i.layerPointToContainerPoint(t),s=this.options.direction,r=e.offsetWidth,a=e.offsetHeight,h=w(this.options.offset),u=this._getAnchor();"top"===s?t=t.add(w(-r/2+h.x,-a+h.y+u.y,!0)):"bottom"===s?t=t.subtract(w(r/2-h.x,-h.y,!0)):"center"===s?t=t.subtract(w(r/2+h.x,a/2-u.y+h.y,!0)):"right"===s||"auto"===s&&o.x<n.x?(s="right",t=t.add(w(h.x+u.x,u.y-a/2+h.y,!0))):(s="left",t=t.subtract(w(r+u.x-h.x,a/2-u.y-h.y,!0))),mt(e,"leaflet-tooltip-right"),mt(e,"leaflet-tooltip-left"),mt(e,"leaflet-tooltip-top"),mt(e,"leaflet-tooltip-bottom"),pt(e,"leaflet-tooltip-"+s),Lt(e,t)},_updatePosition:function(){var t=this._map.latLngToLayerPoint(this._latlng);this._setPosition(t)},setOpacity:function(t){this.options.opacity=t,this._container&&vt(this._container,t)},_animateZoom:function(t){var i=this._map._latLngToNewLayerPoint(this._latlng,t.zoom,t.center);this._setPosition(i)},_getAnchor:function(){return w(this._source&&this._source._getTooltipAnchor&&!this.options.sticky?this._source._getTooltipAnchor():[0,0])}});Le.include({openTooltip:function(t,i,e){return t instanceof ln||(t=new ln(e).setContent(t)),i&&t.setLatLng(i),this.hasLayer(t)?this:this.addLayer(t)},closeTooltip:function(t){return t&&this.removeLayer(t),this}}),Ue.include({bindTooltip:function(t,i){return t instanceof ln?(l(t,i),this._tooltip=t,t._source=this):(this._tooltip&&!i||(this._tooltip=new ln(i,this)),this._tooltip.setContent(t)),this._initTooltipInteractions(),this._tooltip.options.permanent&&this._map&&this._map.hasLayer(this)&&this.openTooltip(),this},unbindTooltip:function(){return this._tooltip&&(this._initTooltipInteractions(!0),this.closeTooltip(),this._tooltip=null),this},_initTooltipInteractions:function(t){if(t||!this._tooltipHandlersAdded){var i=t?"off":"on",e={remove:this.closeTooltip,move:this._moveTooltip};this._tooltip.options.permanent?e.add=this._openTooltip:(e.mouseover=this._openTooltip,e.mouseout=this.closeTooltip,this._tooltip.options.sticky&&(e.mousemove=this._moveTooltip),Vi&&(e.click=this._openTooltip)),this[i](e),this._tooltipHandlersAdded=!t}},openTooltip:function(t,i){if(t instanceof Ue||(i=t,t=this),t instanceof qe)for(var e in this._layers){t=this._layers[e];break}return i||(i=t.getCenter?t.getCenter():t.getLatLng()),this._tooltip&&this._map&&(this._tooltip._source=t,this._tooltip.update(),this._map.openTooltip(this._tooltip,i),this._tooltip.options.interactive&&this._tooltip._container&&(pt(this._tooltip._container,"leaflet-clickable"),this.addInteractiveTarget(this._tooltip._container))),this},closeTooltip:function(){return this._tooltip&&(this._tooltip._close(),this._tooltip.options.interactive&&this._tooltip._container&&(mt(this._tooltip._container,"leaflet-clickable"),this.removeInteractiveTarget(this._tooltip._container))),this},toggleTooltip:function(t){return this._tooltip&&(this._tooltip._map?this.closeTooltip():this.openTooltip(t)),this},isTooltipOpen:function(){return this._tooltip.isOpen()},setTooltipContent:function(t){return this._tooltip&&this._tooltip.setContent(t),this},getTooltip:function(){return this._tooltip},_openTooltip:function(t){var i=t.layer||t.target;this._tooltip&&this._map&&this.openTooltip(i,this._tooltip.options.sticky?t.latlng:void 0)},_moveTooltip:function(t){var i,e,n=t.latlng;this._tooltip.options.sticky&&t.originalEvent&&(i=this._map.mouseEventToContainerPoint(t.originalEvent),e=this._map.containerPointToLayerPoint(i),n=this._map.layerPointToLatLng(e)),this._tooltip.setLatLng(n)}});var cn=Ge.extend({options:{iconSize:[12,12],html:!1,bgPos:null,className:"leaflet-div-icon"},createIcon:function(t){var i=t&&"DIV"===t.tagName?t:document.createElement("div"),e=this.options;if(i.innerHTML=!1!==e.html?e.html:"",e.bgPos){var n=w(e.bgPos);i.style.backgroundPosition=-n.x+"px "+-n.y+"px"}return this._setIconStyles(i,"icon"),i},createShadow:function(){return null}});Ge.Default=Ke;var _n=Ue.extend({options:{tileSize:256,opacity:1,updateWhenIdle:ji,updateWhenZooming:!0,updateInterval:200,zIndex:1,bounds:null,minZoom:0,maxZoom:void 0,maxNativeZoom:void 0,minNativeZoom:void 0,noWrap:!1,pane:"tilePane",className:"",keepBuffer:2},initialize:function(t){l(this,t)},onAdd:function(){this._initContainer(),this._levels={},this._tiles={},this._resetView(),this._update()},beforeAdd:function(t){t._addZoomLimit(this)},onRemove:function(t){this._removeAllTiles(),ut(this._container),t._removeZoomLimit(this),this._container=null,this._tileZoom=void 0},bringToFront:function(){return this._map&&(ct(this._container),this._setAutoZIndex(Math.max)),this},bringToBack:function(){return this._map&&(_t(this._container),this._setAutoZIndex(Math.min)),this},getContainer:function(){return this._container},setOpacity:function(t){return this.options.opacity=t,this._updateOpacity(),this},setZIndex:function(t){return this.options.zIndex=t,this._updateZIndex(),this},isLoading:function(){return this._loading},redraw:function(){return this._map&&(this._removeAllTiles(),this._update()),this},getEvents:function(){var t={viewprereset:this._invalidateAll,viewreset:this._resetView,zoom:this._resetView,moveend:this._onMoveEnd};return this.options.updateWhenIdle||(this._onMove||(this._onMove=o(this._onMoveEnd,this.options.updateInterval,this)),t.move=this._onMove),this._zoomAnimated&&(t.zoomanim=this._animateZoom),t},createTile:function(){return document.createElement("div")},getTileSize:function(){var t=this.options.tileSize;return t instanceof x?t:new x(t,t)},_updateZIndex:function(){this._container&&void 0!==this.options.zIndex&&null!==this.options.zIndex&&(this._container.style.zIndex=this.options.zIndex)},_setAutoZIndex:function(t){for(var i,e=this.getPane().children,n=-t(-1/0,1/0),o=0,s=e.length;o<s;o++)i=e[o].style.zIndex,e[o]!==this._container&&i&&(n=t(n,+i));isFinite(n)&&(this.options.zIndex=n+t(-1,1),this._updateZIndex())},_updateOpacity:function(){if(this._map&&!Li){vt(this._container,this.options.opacity);var t=+new Date,i=!1,e=!1;for(var n in this._tiles){var o=this._tiles[n];if(o.current&&o.loaded){var s=Math.min(1,(t-o.loaded)/200);vt(o.el,s),s<1?i=!0:(o.active?e=!0:this._onOpaqueTile(o),o.active=!0)}}e&&!this._noPrune&&this._pruneTiles(),i&&(g(this._fadeFrame),this._fadeFrame=f(this._updateOpacity,this))}},_onOpaqueTile:r,_initContainer:function(){this._container||(this._container=ht("div","leaflet-layer "+(this.options.className||"")),this._updateZIndex(),this.options.opacity<1&&this._updateOpacity(),this.getPane().appendChild(this._container))},_updateLevels:function(){var t=this._tileZoom,i=this.options.maxZoom;if(void 0!==t){for(var e in this._levels)this._levels[e].el.children.length||e===t?(this._levels[e].el.style.zIndex=i-Math.abs(t-e),this._onUpdateLevel(e)):(ut(this._levels[e].el),this._removeTilesAtZoom(e),this._onRemoveLevel(e),delete this._levels[e]);var n=this._levels[t],o=this._map;return n||((n=this._levels[t]={}).el=ht("div","leaflet-tile-container leaflet-zoom-animated",this._container),n.el.style.zIndex=i,n.origin=o.project(o.unproject(o.getPixelOrigin()),t).round(),n.zoom=t,this._setZoomTransform(n,o.getCenter(),o.getZoom()),n.el.offsetWidth,this._onCreateLevel(n)),this._level=n,n}},_onUpdateLevel:r,_onRemoveLevel:r,_onCreateLevel:r,_pruneTiles:function(){if(this._map){var t,i,e=this._map.getZoom();if(e>this.options.maxZoom||e<this.options.minZoom)this._removeAllTiles();else{for(t in this._tiles)(i=this._tiles[t]).retain=i.current;for(t in this._tiles)if((i=this._tiles[t]).current&&!i.active){var n=i.coords;this._retainParent(n.x,n.y,n.z,n.z-5)||this._retainChildren(n.x,n.y,n.z,n.z+2)}for(t in this._tiles)this._tiles[t].retain||this._removeTile(t)}}},_removeTilesAtZoom:function(t){for(var i in this._tiles)this._tiles[i].coords.z===t&&this._removeTile(i)},_removeAllTiles:function(){for(var t in this._tiles)this._removeTile(t)},_invalidateAll:function(){for(var t in this._levels)ut(this._levels[t].el),this._onRemoveLevel(t),delete this._levels[t];this._removeAllTiles(),this._tileZoom=void 0},_retainParent:function(t,i,e,n){var o=Math.floor(t/2),s=Math.floor(i/2),r=e-1,a=new x(+o,+s);a.z=+r;var h=this._tileCoordsToKey(a),u=this._tiles[h];return u&&u.active?(u.retain=!0,!0):(u&&u.loaded&&(u.retain=!0),r>n&&this._retainParent(o,s,r,n))},_retainChildren:function(t,i,e,n){for(var o=2*t;o<2*t+2;o++)for(var s=2*i;s<2*i+2;s++){var r=new x(o,s);r.z=e+1;var a=this._tileCoordsToKey(r),h=this._tiles[a];h&&h.active?h.retain=!0:(h&&h.loaded&&(h.retain=!0),e+1<n&&this._retainChildren(o,s,e+1,n))}},_resetView:function(t){var i=t&&(t.pinch||t.flyTo);this._setView(this._map.getCenter(),this._map.getZoom(),i,i)},_animateZoom:function(t){this._setView(t.center,t.zoom,!0,t.noUpdate)},_clampZoom:function(t){var i=this.options;return void 0!==i.minNativeZoom&&t<i.minNativeZoom?i.minNativeZoom:void 0!==i.maxNativeZoom&&i.maxNativeZoom<t?i.maxNativeZoom:t},_setView:function(t,i,e,n){var o=this._clampZoom(Math.round(i));(void 0!==this.options.maxZoom&&o>this.options.maxZoom||void 0!==this.options.minZoom&&o<this.options.minZoom)&&(o=void 0);var s=this.options.updateWhenZooming&&o!==this._tileZoom;n&&!s||(this._tileZoom=o,this._abortLoading&&this._abortLoading(),this._updateLevels(),this._resetGrid(),void 0!==o&&this._update(t),e||this._pruneTiles(),this._noPrune=!!e),this._setZoomTransforms(t,i)},_setZoomTransforms:function(t,i){for(var e in this._levels)this._setZoomTransform(this._levels[e],t,i)},_setZoomTransform:function(t,i,e){var n=this._map.getZoomScale(e,t.zoom),o=t.origin.multiplyBy(n).subtract(this._map._getNewPixelOrigin(i,e)).round();Ni?wt(t.el,o,n):Lt(t.el,o)},_resetGrid:function(){var t=this._map,i=t.options.crs,e=this._tileSize=this.getTileSize(),n=this._tileZoom,o=this._map.getPixelWorldBounds(this._tileZoom);o&&(this._globalTileRange=this._pxBoundsToTileRange(o)),this._wrapX=i.wrapLng&&!this.options.noWrap&&[Math.floor(t.project([0,i.wrapLng[0]],n).x/e.x),Math.ceil(t.project([0,i.wrapLng[1]],n).x/e.y)],this._wrapY=i.wrapLat&&!this.options.noWrap&&[Math.floor(t.project([i.wrapLat[0],0],n).y/e.x),Math.ceil(t.project([i.wrapLat[1],0],n).y/e.y)]},_onMoveEnd:function(){this._map&&!this._map._animatingZoom&&this._update()},_getTiledPixelBounds:function(t){var i=this._map,e=i._animatingZoom?Math.max(i._animateToZoom,i.getZoom()):i.getZoom(),n=i.getZoomScale(e,this._tileZoom),o=i.project(t,this._tileZoom).floor(),s=i.getSize().divideBy(2*n);return new P(o.subtract(s),o.add(s))},_update:function(t){var i=this._map;if(i){var e=this._clampZoom(i.getZoom());if(void 0===t&&(t=i.getCenter()),void 0!==this._tileZoom){var n=this._getTiledPixelBounds(t),o=this._pxBoundsToTileRange(n),s=o.getCenter(),r=[],a=this.options.keepBuffer,h=new P(o.getBottomLeft().subtract([a,-a]),o.getTopRight().add([a,-a]));if(!(isFinite(o.min.x)&&isFinite(o.min.y)&&isFinite(o.max.x)&&isFinite(o.max.y)))throw new Error("Attempted to load an infinite number of tiles");for(var u in this._tiles){var l=this._tiles[u].coords;l.z===this._tileZoom&&h.contains(new x(l.x,l.y))||(this._tiles[u].current=!1)}if(Math.abs(e-this._tileZoom)>1)this._setView(t,e);else{for(var c=o.min.y;c<=o.max.y;c++)for(var _=o.min.x;_<=o.max.x;_++){var d=new x(_,c);if(d.z=this._tileZoom,this._isValidTile(d)){var p=this._tiles[this._tileCoordsToKey(d)];p?p.current=!0:r.push(d)}}if(r.sort(function(t,i){return t.distanceTo(s)-i.distanceTo(s)}),0!==r.length){this._loading||(this._loading=!0,this.fire("loading"));var m=document.createDocumentFragment();for(_=0;_<r.length;_++)this._addTile(r[_],m);this._level.el.appendChild(m)}}}}},_isValidTile:function(t){var i=this._map.options.crs;if(!i.infinite){var e=this._globalTileRange;if(!i.wrapLng&&(t.x<e.min.x||t.x>e.max.x)||!i.wrapLat&&(t.y<e.min.y||t.y>e.max.y))return!1}if(!this.options.bounds)return!0;var n=this._tileCoordsToBounds(t);return z(this.options.bounds).overlaps(n)},_keyToBounds:function(t){return this._tileCoordsToBounds(this._keyToTileCoords(t))},_tileCoordsToNwSe:function(t){var i=this._map,e=this.getTileSize(),n=t.scaleBy(e),o=n.add(e);return[i.unproject(n,t.z),i.unproject(o,t.z)]},_tileCoordsToBounds:function(t){var i=this._tileCoordsToNwSe(t),e=new T(i[0],i[1]);return this.options.noWrap||(e=this._map.wrapLatLngBounds(e)),e},_tileCoordsToKey:function(t){return t.x+":"+t.y+":"+t.z},_keyToTileCoords:function(t){var i=t.split(":"),e=new x(+i[0],+i[1]);return e.z=+i[2],e},_removeTile:function(t){var i=this._tiles[t];i&&(Ci||i.el.setAttribute("src",ni),ut(i.el),delete this._tiles[t],this.fire("tileunload",{tile:i.el,coords:this._keyToTileCoords(t)}))},_initTile:function(t){pt(t,"leaflet-tile");var i=this.getTileSize();t.style.width=i.x+"px",t.style.height=i.y+"px",t.onselectstart=r,t.onmousemove=r,Li&&this.options.opacity<1&&vt(t,this.options.opacity),Ti&&!zi&&(t.style.WebkitBackfaceVisibility="hidden")},_addTile:function(t,i){var n=this._getTilePos(t),o=this._tileCoordsToKey(t),s=this.createTile(this._wrapCoords(t),e(this._tileReady,this,t));this._initTile(s),this.createTile.length<2&&f(e(this._tileReady,this,t,null,s)),Lt(s,n),this._tiles[o]={el:s,coords:t,current:!0},i.appendChild(s),this.fire("tileloadstart",{tile:s,coords:t})},_tileReady:function(t,i,n){if(this._map){i&&this.fire("tileerror",{error:i,tile:n,coords:t});var o=this._tileCoordsToKey(t);(n=this._tiles[o])&&(n.loaded=+new Date,this._map._fadeAnimated?(vt(n.el,0),g(this._fadeFrame),this._fadeFrame=f(this._updateOpacity,this)):(n.active=!0,this._pruneTiles()),i||(pt(n.el,"leaflet-tile-loaded"),this.fire("tileload",{tile:n.el,coords:t})),this._noTilesToLoad()&&(this._loading=!1,this.fire("load"),Li||!this._map._fadeAnimated?f(this._pruneTiles,this):setTimeout(e(this._pruneTiles,this),250)))}},_getTilePos:function(t){return t.scaleBy(this.getTileSize()).subtract(this._level.origin)},_wrapCoords:function(t){var i=new x(this._wrapX?s(t.x,this._wrapX):t.x,this._wrapY?s(t.y,this._wrapY):t.y);return i.z=t.z,i},_pxBoundsToTileRange:function(t){var i=this.getTileSize();return new P(t.min.unscaleBy(i).floor(),t.max.unscaleBy(i).ceil().subtract([1,1]))},_noTilesToLoad:function(){for(var t in this._tiles)if(!this._tiles[t].loaded)return!1;return!0}}),dn=_n.extend({options:{minZoom:0,maxZoom:18,subdomains:"abc",errorTileUrl:"",zoomOffset:0,tms:!1,zoomReverse:!1,detectRetina:!1,crossOrigin:!1},initialize:function(t,i){this._url=t,(i=l(this,i)).detectRetina&&Ki&&i.maxZoom>0&&(i.tileSize=Math.floor(i.tileSize/2),i.zoomReverse?(i.zoomOffset--,i.minZoom++):(i.zoomOffset++,i.maxZoom--),i.minZoom=Math.max(0,i.minZoom)),"string"==typeof i.subdomains&&(i.subdomains=i.subdomains.split("")),Ti||this.on("tileunload",this._onTileRemove)},setUrl:function(t,i){return this._url=t,i||this.redraw(),this},createTile:function(t,i){var n=document.createElement("img");return V(n,"load",e(this._tileOnLoad,this,i,n)),V(n,"error",e(this._tileOnError,this,i,n)),this.options.crossOrigin&&(n.crossOrigin=""),n.alt="",n.setAttribute("role","presentation"),n.src=this.getTileUrl(t),n},getTileUrl:function(t){var e={r:Ki?"@2x":"",s:this._getSubdomain(t),x:t.x,y:t.y,z:this._getZoomForUrl()};if(this._map&&!this._map.options.crs.infinite){var n=this._globalTileRange.max.y-t.y;this.options.tms&&(e.y=n),e["-y"]=n}return _(this._url,i(e,this.options))},_tileOnLoad:function(t,i){Li?setTimeout(e(t,this,null,i),0):t(null,i)},_tileOnError:function(t,i,e){var n=this.options.errorTileUrl;n&&i.getAttribute("src")!==n&&(i.src=n),t(e,i)},_onTileRemove:function(t){t.tile.onload=null},_getZoomForUrl:function(){var t=this._tileZoom,i=this.options.maxZoom,e=this.options.zoomReverse,n=this.options.zoomOffset;return e&&(t=i-t),t+n},_getSubdomain:function(t){var i=Math.abs(t.x+t.y)%this.options.subdomains.length;return this.options.subdomains[i]},_abortLoading:function(){var t,i;for(t in this._tiles)this._tiles[t].coords.z!==this._tileZoom&&((i=this._tiles[t].el).onload=r,i.onerror=r,i.complete||(i.src=ni,ut(i),delete this._tiles[t]))}}),pn=dn.extend({defaultWmsParams:{service:"WMS",request:"GetMap",layers:"",styles:"",format:"image/jpeg",transparent:!1,version:"1.1.1"},options:{crs:null,uppercase:!1},initialize:function(t,e){this._url=t;var n=i({},this.defaultWmsParams);for(var o in e)o in this.options||(n[o]=e[o]);var s=(e=l(this,e)).detectRetina&&Ki?2:1,r=this.getTileSize();n.width=r.x*s,n.height=r.y*s,this.wmsParams=n},onAdd:function(t){this._crs=this.options.crs||t.options.crs,this._wmsVersion=parseFloat(this.wmsParams.version);var i=this._wmsVersion>=1.3?"crs":"srs";this.wmsParams[i]=this._crs.code,dn.prototype.onAdd.call(this,t)},getTileUrl:function(t){var i=this._tileCoordsToNwSe(t),e=this._crs,n=b(e.project(i[0]),e.project(i[1])),o=n.min,s=n.max,r=(this._wmsVersion>=1.3&&this._crs===He?[o.y,o.x,s.y,s.x]:[o.x,o.y,s.x,s.y]).join(","),a=L.TileLayer.prototype.getTileUrl.call(this,t);return a+c(this.wmsParams,a,this.options.uppercase)+(this.options.uppercase?"&BBOX=":"&bbox=")+r},setParams:function(t,e){return i(this.wmsParams,t),e||this.redraw(),this}});dn.WMS=pn,Yt.wms=function(t,i){return new pn(t,i)};var mn=Ue.extend({options:{padding:.1,tolerance:0},initialize:function(t){l(this,t),n(this),this._layers=this._layers||{}},onAdd:function(){this._container||(this._initContainer(),this._zoomAnimated&&pt(this._container,"leaflet-zoom-animated")),this.getPane().appendChild(this._container),this._update(),this.on("update",this._updatePaths,this)},onRemove:function(){this.off("update",this._updatePaths,this),this._destroyContainer()},getEvents:function(){var t={viewreset:this._reset,zoom:this._onZoom,moveend:this._update,zoomend:this._onZoomEnd};return this._zoomAnimated&&(t.zoomanim=this._onAnimZoom),t},_onAnimZoom:function(t){this._updateTransform(t.center,t.zoom)},_onZoom:function(){this._updateTransform(this._map.getCenter(),this._map.getZoom())},_updateTransform:function(t,i){var e=this._map.getZoomScale(i,this._zoom),n=Pt(this._container),o=this._map.getSize().multiplyBy(.5+this.options.padding),s=this._map.project(this._center,i),r=this._map.project(t,i).subtract(s),a=o.multiplyBy(-e).add(n).add(o).subtract(r);Ni?wt(this._container,a,e):Lt(this._container,a)},_reset:function(){this._update(),this._updateTransform(this._center,this._zoom);for(var t in this._layers)this._layers[t]._reset()},_onZoomEnd:function(){for(var t in this._layers)this._layers[t]._project()},_updatePaths:function(){for(var t in this._layers)this._layers[t]._update()},_update:function(){var t=this.options.padding,i=this._map.getSize(),e=this._map.containerPointToLayerPoint(i.multiplyBy(-t)).round();this._bounds=new P(e,e.add(i.multiplyBy(1+2*t)).round()),this._center=this._map.getCenter(),this._zoom=this._map.getZoom()}}),fn=mn.extend({getEvents:function(){var t=mn.prototype.getEvents.call(this);return t.viewprereset=this._onViewPreReset,t},_onViewPreReset:function(){this._postponeUpdatePaths=!0},onAdd:function(){mn.prototype.onAdd.call(this),this._draw()},_initContainer:function(){var t=this._container=document.createElement("canvas");V(t,"mousemove",o(this._onMouseMove,32,this),this),V(t,"click dblclick mousedown mouseup contextmenu",this._onClick,this),V(t,"mouseout",this._handleMouseOut,this),this._ctx=t.getContext("2d")},_destroyContainer:function(){delete this._ctx,ut(this._container),q(this._container),delete this._container},_updatePaths:function(){if(!this._postponeUpdatePaths){this._redrawBounds=null;for(var t in this._layers)this._layers[t]._update();this._redraw()}},_update:function(){if(!this._map._animatingZoom||!this._bounds){this._drawnLayers={},mn.prototype._update.call(this);var t=this._bounds,i=this._container,e=t.getSize(),n=Ki?2:1;Lt(i,t.min),i.width=n*e.x,i.height=n*e.y,i.style.width=e.x+"px",i.style.height=e.y+"px",Ki&&this._ctx.scale(2,2),this._ctx.translate(-t.min.x,-t.min.y),this.fire("update")}},_reset:function(){mn.prototype._reset.call(this),this._postponeUpdatePaths&&(this._postponeUpdatePaths=!1,this._updatePaths())},_initPath:function(t){this._updateDashArray(t),this._layers[n(t)]=t;var i=t._order={layer:t,prev:this._drawLast,next:null};this._drawLast&&(this._drawLast.next=i),this._drawLast=i,this._drawFirst=this._drawFirst||this._drawLast},_addPath:function(t){this._requestRedraw(t)},_removePath:function(t){var i=t._order,e=i.next,n=i.prev;e?e.prev=n:this._drawLast=n,n?n.next=e:this._drawFirst=e,delete t._order,delete this._layers[L.stamp(t)],this._requestRedraw(t)},_updatePath:function(t){this._extendRedrawBounds(t),t._project(),t._update(),this._requestRedraw(t)},_updateStyle:function(t){this._updateDashArray(t),this._requestRedraw(t)},_updateDashArray:function(t){if(t.options.dashArray){var i,e=t.options.dashArray.split(","),n=[];for(i=0;i<e.length;i++)n.push(Number(e[i]));t.options._dashArray=n}},_requestRedraw:function(t){this._map&&(this._extendRedrawBounds(t),this._redrawRequest=this._redrawRequest||f(this._redraw,this))},_extendRedrawBounds:function(t){if(t._pxBounds){var i=(t.options.weight||0)+1;this._redrawBounds=this._redrawBounds||new P,this._redrawBounds.extend(t._pxBounds.min.subtract([i,i])),this._redrawBounds.extend(t._pxBounds.max.add([i,i]))}},_redraw:function(){this._redrawRequest=null,this._redrawBounds&&(this._redrawBounds.min._floor(),this._redrawBounds.max._ceil()),this._clear(),this._draw(),this._redrawBounds=null},_clear:function(){var t=this._redrawBounds;if(t){var i=t.getSize();this._ctx.clearRect(t.min.x,t.min.y,i.x,i.y)}else this._ctx.clearRect(0,0,this._container.width,this._container.height)},_draw:function(){var t,i=this._redrawBounds;if(this._ctx.save(),i){var e=i.getSize();this._ctx.beginPath(),this._ctx.rect(i.min.x,i.min.y,e.x,e.y),this._ctx.clip()}this._drawing=!0;for(var n=this._drawFirst;n;n=n.next)t=n.layer,(!i||t._pxBounds&&t._pxBounds.intersects(i))&&t._updatePath();this._drawing=!1,this._ctx.restore()},_updatePoly:function(t,i){if(this._drawing){var e,n,o,s,r=t._parts,a=r.length,h=this._ctx;if(a){for(this._drawnLayers[t._leaflet_id]=t,h.beginPath(),e=0;e<a;e++){for(n=0,o=r[e].length;n<o;n++)s=r[e][n],h[n?"lineTo":"moveTo"](s.x,s.y);i&&h.closePath()}this._fillStroke(h,t)}}},_updateCircle:function(t){if(this._drawing&&!t._empty()){var i=t._point,e=this._ctx,n=Math.max(Math.round(t._radius),1),o=(Math.max(Math.round(t._radiusY),1)||n)/n;this._drawnLayers[t._leaflet_id]=t,1!==o&&(e.save(),e.scale(1,o)),e.beginPath(),e.arc(i.x,i.y/o,n,0,2*Math.PI,!1),1!==o&&e.restore(),this._fillStroke(e,t)}},_fillStroke:function(t,i){var e=i.options;e.fill&&(t.globalAlpha=e.fillOpacity,t.fillStyle=e.fillColor||e.color,t.fill(e.fillRule||"evenodd")),e.stroke&&0!==e.weight&&(t.setLineDash&&t.setLineDash(i.options&&i.options._dashArray||[]),t.globalAlpha=e.opacity,t.lineWidth=e.weight,t.strokeStyle=e.color,t.lineCap=e.lineCap,t.lineJoin=e.lineJoin,t.stroke())},_onClick:function(t){for(var i,e,n=this._map.mouseEventToLayerPoint(t),o=this._drawFirst;o;o=o.next)(i=o.layer).options.interactive&&i._containsPoint(n)&&!this._map._draggableMoved(i)&&(e=i);e&&(et(t),this._fireEvent([e],t))},_onMouseMove:function(t){if(this._map&&!this._map.dragging.moving()&&!this._map._animatingZoom){var i=this._map.mouseEventToLayerPoint(t);this._handleMouseHover(t,i)}},_handleMouseOut:function(t){var i=this._hoveredLayer;i&&(mt(this._container,"leaflet-interactive"),this._fireEvent([i],t,"mouseout"),this._hoveredLayer=null)},_handleMouseHover:function(t,i){for(var e,n,o=this._drawFirst;o;o=o.next)(e=o.layer).options.interactive&&e._containsPoint(i)&&(n=e);n!==this._hoveredLayer&&(this._handleMouseOut(t),n&&(pt(this._container,"leaflet-interactive"),this._fireEvent([n],t,"mouseover"),this._hoveredLayer=n)),this._hoveredLayer&&this._fireEvent([this._hoveredLayer],t)},_fireEvent:function(t,i,e){this._map._fireDOMEvent(i,e||i.type,t)},_bringToFront:function(t){var i=t._order,e=i.next,n=i.prev;e&&(e.prev=n,n?n.next=e:e&&(this._drawFirst=e),i.prev=this._drawLast,this._drawLast.next=i,i.next=null,this._drawLast=i,this._requestRedraw(t))},_bringToBack:function(t){var i=t._order,e=i.next,n=i.prev;n&&(n.next=e,e?e.prev=n:n&&(this._drawLast=n),i.prev=null,i.next=this._drawFirst,this._drawFirst.prev=i,this._drawFirst=i,this._requestRedraw(t))}}),gn=function(){try{return document.namespaces.add("lvml","urn:schemas-microsoft-com:vml"),function(t){return document.createElement("<lvml:"+t+' class="lvml">')}}catch(t){return function(t){return document.createElement("<"+t+' xmlns="urn:schemas-microsoft.com:vml" class="lvml">')}}}(),vn={_initContainer:function(){this._container=ht("div","leaflet-vml-container")},_update:function(){this._map._animatingZoom||(mn.prototype._update.call(this),this.fire("update"))},_initPath:function(t){var i=t._container=gn("shape");pt(i,"leaflet-vml-shape "+(this.options.className||"")),i.coordsize="1 1",t._path=gn("path"),i.appendChild(t._path),this._updateStyle(t),this._layers[n(t)]=t},_addPath:function(t){var i=t._container;this._container.appendChild(i),t.options.interactive&&t.addInteractiveTarget(i)},_removePath:function(t){var i=t._container;ut(i),t.removeInteractiveTarget(i),delete this._layers[n(t)]},_updateStyle:function(t){var i=t._stroke,e=t._fill,n=t.options,o=t._container;o.stroked=!!n.stroke,o.filled=!!n.fill,n.stroke?(i||(i=t._stroke=gn("stroke")),o.appendChild(i),i.weight=n.weight+"px",i.color=n.color,i.opacity=n.opacity,n.dashArray?i.dashStyle=ei(n.dashArray)?n.dashArray.join(" "):n.dashArray.replace(/( *, *)/g," "):i.dashStyle="",i.endcap=n.lineCap.replace("butt","flat"),i.joinstyle=n.lineJoin):i&&(o.removeChild(i),t._stroke=null),n.fill?(e||(e=t._fill=gn("fill")),o.appendChild(e),e.color=n.fillColor||n.color,e.opacity=n.fillOpacity):e&&(o.removeChild(e),t._fill=null)},_updateCircle:function(t){var i=t._point.round(),e=Math.round(t._radius),n=Math.round(t._radiusY||e);this._setPath(t,t._empty()?"M0 0":"AL "+i.x+","+i.y+" "+e+","+n+" 0,23592600")},_setPath:function(t,i){t._path.v=i},_bringToFront:function(t){ct(t._container)},_bringToBack:function(t){_t(t._container)}},yn=Ji?gn:E,xn=mn.extend({getEvents:function(){var t=mn.prototype.getEvents.call(this);return t.zoomstart=this._onZoomStart,t},_initContainer:function(){this._container=yn("svg"),this._container.setAttribute("pointer-events","none"),this._rootGroup=yn("g"),this._container.appendChild(this._rootGroup)},_destroyContainer:function(){ut(this._container),q(this._container),delete this._container,delete this._rootGroup,delete this._svgSize},_onZoomStart:function(){this._update()},_update:function(){if(!this._map._animatingZoom||!this._bounds){mn.prototype._update.call(this);var t=this._bounds,i=t.getSize(),e=this._container;this._svgSize&&this._svgSize.equals(i)||(this._svgSize=i,e.setAttribute("width",i.x),e.setAttribute("height",i.y)),Lt(e,t.min),e.setAttribute("viewBox",[t.min.x,t.min.y,i.x,i.y].join(" ")),this.fire("update")}},_initPath:function(t){var i=t._path=yn("path");t.options.className&&pt(i,t.options.className),t.options.interactive&&pt(i,"leaflet-interactive"),this._updateStyle(t),this._layers[n(t)]=t},_addPath:function(t){this._rootGroup||this._initContainer(),this._rootGroup.appendChild(t._path),t.addInteractiveTarget(t._path)},_removePath:function(t){ut(t._path),t.removeInteractiveTarget(t._path),delete this._layers[n(t)]},_updatePath:function(t){t._project(),t._update()},_updateStyle:function(t){var i=t._path,e=t.options;i&&(e.stroke?(i.setAttribute("stroke",e.color),i.setAttribute("stroke-opacity",e.opacity),i.setAttribute("stroke-width",e.weight),i.setAttribute("stroke-linecap",e.lineCap),i.setAttribute("stroke-linejoin",e.lineJoin),e.dashArray?i.setAttribute("stroke-dasharray",e.dashArray):i.removeAttribute("stroke-dasharray"),e.dashOffset?i.setAttribute("stroke-dashoffset",e.dashOffset):i.removeAttribute("stroke-dashoffset")):i.setAttribute("stroke","none"),e.fill?(i.setAttribute("fill",e.fillColor||e.color),i.setAttribute("fill-opacity",e.fillOpacity),i.setAttribute("fill-rule",e.fillRule||"evenodd")):i.setAttribute("fill","none"))},_updatePoly:function(t,i){this._setPath(t,k(t._parts,i))},_updateCircle:function(t){var i=t._point,e=Math.max(Math.round(t._radius),1),n="a"+e+","+(Math.max(Math.round(t._radiusY),1)||e)+" 0 1,0 ",o=t._empty()?"M0 0":"M"+(i.x-e)+","+i.y+n+2*e+",0 "+n+2*-e+",0 ";this._setPath(t,o)},_setPath:function(t,i){t._path.setAttribute("d",i)},_bringToFront:function(t){ct(t._path)},_bringToBack:function(t){_t(t._path)}});Ji&&xn.include(vn),Le.include({getRenderer:function(t){var i=t.options.renderer||this._getPaneRenderer(t.options.pane)||this.options.renderer||this._renderer;return i||(i=this._renderer=this.options.preferCanvas&&Xt()||Jt()),this.hasLayer(i)||this.addLayer(i),i},_getPaneRenderer:function(t){if("overlayPane"===t||void 0===t)return!1;var i=this._paneRenderers[t];return void 0===i&&(i=xn&&Jt({pane:t})||fn&&Xt({pane:t}),this._paneRenderers[t]=i),i}});var wn=en.extend({initialize:function(t,i){en.prototype.initialize.call(this,this._boundsToLatLngs(t),i)},setBounds:function(t){return this.setLatLngs(this._boundsToLatLngs(t))},_boundsToLatLngs:function(t){return t=z(t),[t.getSouthWest(),t.getNorthWest(),t.getNorthEast(),t.getSouthEast()]}});xn.create=yn,xn.pointsToPath=k,nn.geometryToLayer=Wt,nn.coordsToLatLng=Ht,nn.coordsToLatLngs=Ft,nn.latLngToCoords=Ut,nn.latLngsToCoords=Vt,nn.getFeature=qt,nn.asFeature=Gt,Le.mergeOptions({boxZoom:!0});var Ln=Ze.extend({initialize:function(t){this._map=t,this._container=t._container,this._pane=t._panes.overlayPane,this._resetStateTimeout=0,t.on("unload",this._destroy,this)},addHooks:function(){V(this._container,"mousedown",this._onMouseDown,this)},removeHooks:function(){q(this._container,"mousedown",this._onMouseDown,this)},moved:function(){return this._moved},_destroy:function(){ut(this._pane),delete this._pane},_resetState:function(){this._resetStateTimeout=0,this._moved=!1},_clearDeferredResetState:function(){0!==this._resetStateTimeout&&(clearTimeout(this._resetStateTimeout),this._resetStateTimeout=0)},_onMouseDown:function(t){if(!t.shiftKey||1!==t.which&&1!==t.button)return!1;this._clearDeferredResetState(),this._resetState(),mi(),bt(),this._startPoint=this._map.mouseEventToContainerPoint(t),V(document,{contextmenu:Q,mousemove:this._onMouseMove,mouseup:this._onMouseUp,keydown:this._onKeyDown},this)},_onMouseMove:function(t){this._moved||(this._moved=!0,this._box=ht("div","leaflet-zoom-box",this._container),pt(this._container,"leaflet-crosshair"),this._map.fire("boxzoomstart")),this._point=this._map.mouseEventToContainerPoint(t);var i=new P(this._point,this._startPoint),e=i.getSize();Lt(this._box,i.min),this._box.style.width=e.x+"px",this._box.style.height=e.y+"px"},_finish:function(){this._moved&&(ut(this._box),mt(this._container,"leaflet-crosshair")),fi(),Tt(),q(document,{contextmenu:Q,mousemove:this._onMouseMove,mouseup:this._onMouseUp,keydown:this._onKeyDown},this)},_onMouseUp:function(t){if((1===t.which||1===t.button)&&(this._finish(),this._moved)){this._clearDeferredResetState(),this._resetStateTimeout=setTimeout(e(this._resetState,this),0);var i=new T(this._map.containerPointToLatLng(this._startPoint),this._map.containerPointToLatLng(this._point));this._map.fitBounds(i).fire("boxzoomend",{boxZoomBounds:i})}},_onKeyDown:function(t){27===t.keyCode&&this._finish()}});Le.addInitHook("addHandler","boxZoom",Ln),Le.mergeOptions({doubleClickZoom:!0});var Pn=Ze.extend({addHooks:function(){this._map.on("dblclick",this._onDoubleClick,this)},removeHooks:function(){this._map.off("dblclick",this._onDoubleClick,this)},_onDoubleClick:function(t){var i=this._map,e=i.getZoom(),n=i.options.zoomDelta,o=t.originalEvent.shiftKey?e-n:e+n;"center"===i.options.doubleClickZoom?i.setZoom(o):i.setZoomAround(t.containerPoint,o)}});Le.addInitHook("addHandler","doubleClickZoom",Pn),Le.mergeOptions({dragging:!0,inertia:!zi,inertiaDeceleration:3400,inertiaMaxSpeed:1/0,easeLinearity:.2,worldCopyJump:!1,maxBoundsViscosity:0});var bn=Ze.extend({addHooks:function(){if(!this._draggable){var t=this._map;this._draggable=new Be(t._mapPane,t._container),this._draggable.on({dragstart:this._onDragStart,drag:this._onDrag,dragend:this._onDragEnd},this),this._draggable.on("predrag",this._onPreDragLimit,this),t.options.worldCopyJump&&(this._draggable.on("predrag",this._onPreDragWrap,this),t.on("zoomend",this._onZoomEnd,this),t.whenReady(this._onZoomEnd,this))}pt(this._map._container,"leaflet-grab leaflet-touch-drag"),this._draggable.enable(),this._positions=[],this._times=[]},removeHooks:function(){mt(this._map._container,"leaflet-grab"),mt(this._map._container,"leaflet-touch-drag"),this._draggable.disable()},moved:function(){return this._draggable&&this._draggable._moved},moving:function(){return this._draggable&&this._draggable._moving},_onDragStart:function(){var t=this._map;if(t._stop(),this._map.options.maxBounds&&this._map.options.maxBoundsViscosity){var i=z(this._map.options.maxBounds);this._offsetLimit=b(this._map.latLngToContainerPoint(i.getNorthWest()).multiplyBy(-1),this._map.latLngToContainerPoint(i.getSouthEast()).multiplyBy(-1).add(this._map.getSize())),this._viscosity=Math.min(1,Math.max(0,this._map.options.maxBoundsViscosity))}else this._offsetLimit=null;t.fire("movestart").fire("dragstart"),t.options.inertia&&(this._positions=[],this._times=[])},_onDrag:function(t){if(this._map.options.inertia){var i=this._lastTime=+new Date,e=this._lastPos=this._draggable._absPos||this._draggable._newPos;this._positions.push(e),this._times.push(i),this._prunePositions(i)}this._map.fire("move",t).fire("drag",t)},_prunePositions:function(t){for(;this._positions.length>1&&t-this._times[0]>50;)this._positions.shift(),this._times.shift()},_onZoomEnd:function(){var t=this._map.getSize().divideBy(2),i=this._map.latLngToLayerPoint([0,0]);this._initialWorldOffset=i.subtract(t).x,this._worldWidth=this._map.getPixelWorldBounds().getSize().x},_viscousLimit:function(t,i){return t-(t-i)*this._viscosity},_onPreDragLimit:function(){if(this._viscosity&&this._offsetLimit){var t=this._draggable._newPos.subtract(this._draggable._startPos),i=this._offsetLimit;t.x<i.min.x&&(t.x=this._viscousLimit(t.x,i.min.x)),t.y<i.min.y&&(t.y=this._viscousLimit(t.y,i.min.y)),t.x>i.max.x&&(t.x=this._viscousLimit(t.x,i.max.x)),t.y>i.max.y&&(t.y=this._viscousLimit(t.y,i.max.y)),this._draggable._newPos=this._draggable._startPos.add(t)}},_onPreDragWrap:function(){var t=this._worldWidth,i=Math.round(t/2),e=this._initialWorldOffset,n=this._draggable._newPos.x,o=(n-i+e)%t+i-e,s=(n+i+e)%t-i-e,r=Math.abs(o+e)<Math.abs(s+e)?o:s;this._draggable._absPos=this._draggable._newPos.clone(),this._draggable._newPos.x=r},_onDragEnd:function(t){var i=this._map,e=i.options,n=!e.inertia||this._times.length<2;if(i.fire("dragend",t),n)i.fire("moveend");else{this._prunePositions(+new Date);var o=this._lastPos.subtract(this._positions[0]),s=(this._lastTime-this._times[0])/1e3,r=e.easeLinearity,a=o.multiplyBy(r/s),h=a.distanceTo([0,0]),u=Math.min(e.inertiaMaxSpeed,h),l=a.multiplyBy(u/h),c=u/(e.inertiaDeceleration*r),_=l.multiplyBy(-c/2).round();_.x||_.y?(_=i._limitOffset(_,i.options.maxBounds),f(function(){i.panBy(_,{duration:c,easeLinearity:r,noMoveStart:!0,animate:!0})})):i.fire("moveend")}}});Le.addInitHook("addHandler","dragging",bn),Le.mergeOptions({keyboard:!0,keyboardPanDelta:80});var Tn=Ze.extend({keyCodes:{left:[37],right:[39],down:[40],up:[38],zoomIn:[187,107,61,171],zoomOut:[189,109,54,173]},initialize:function(t){this._map=t,this._setPanDelta(t.options.keyboardPanDelta),this._setZoomDelta(t.options.zoomDelta)},addHooks:function(){var t=this._map._container;t.tabIndex<=0&&(t.tabIndex="0"),V(t,{focus:this._onFocus,blur:this._onBlur,mousedown:this._onMouseDown},this),this._map.on({focus:this._addHooks,blur:this._removeHooks},this)},removeHooks:function(){this._removeHooks(),q(this._map._container,{focus:this._onFocus,blur:this._onBlur,mousedown:this._onMouseDown},this),this._map.off({focus:this._addHooks,blur:this._removeHooks},this)},_onMouseDown:function(){if(!this._focused){var t=document.body,i=document.documentElement,e=t.scrollTop||i.scrollTop,n=t.scrollLeft||i.scrollLeft;this._map._container.focus(),window.scrollTo(n,e)}},_onFocus:function(){this._focused=!0,this._map.fire("focus")},_onBlur:function(){this._focused=!1,this._map.fire("blur")},_setPanDelta:function(t){var i,e,n=this._panKeys={},o=this.keyCodes;for(i=0,e=o.left.length;i<e;i++)n[o.left[i]]=[-1*t,0];for(i=0,e=o.right.length;i<e;i++)n[o.right[i]]=[t,0];for(i=0,e=o.down.length;i<e;i++)n[o.down[i]]=[0,t];for(i=0,e=o.up.length;i<e;i++)n[o.up[i]]=[0,-1*t]},_setZoomDelta:function(t){var i,e,n=this._zoomKeys={},o=this.keyCodes;for(i=0,e=o.zoomIn.length;i<e;i++)n[o.zoomIn[i]]=t;for(i=0,e=o.zoomOut.length;i<e;i++)n[o.zoomOut[i]]=-t},_addHooks:function(){V(document,"keydown",this._onKeyDown,this)},_removeHooks:function(){q(document,"keydown",this._onKeyDown,this)},_onKeyDown:function(t){if(!(t.altKey||t.ctrlKey||t.metaKey)){var i,e=t.keyCode,n=this._map;if(e in this._panKeys){if(n._panAnim&&n._panAnim._inProgress)return;i=this._panKeys[e],t.shiftKey&&(i=w(i).multiplyBy(3)),n.panBy(i),n.options.maxBounds&&n.panInsideBounds(n.options.maxBounds)}else if(e in this._zoomKeys)n.setZoom(n.getZoom()+(t.shiftKey?3:1)*this._zoomKeys[e]);else{if(27!==e||!n._popup||!n._popup.options.closeOnEscapeKey)return;n.closePopup()}Q(t)}}});Le.addInitHook("addHandler","keyboard",Tn),Le.mergeOptions({scrollWheelZoom:!0,wheelDebounceTime:40,wheelPxPerZoomLevel:60});var zn=Ze.extend({addHooks:function(){V(this._map._container,"mousewheel",this._onWheelScroll,this),this._delta=0},removeHooks:function(){q(this._map._container,"mousewheel",this._onWheelScroll,this)},_onWheelScroll:function(t){var i=it(t),n=this._map.options.wheelDebounceTime;this._delta+=i,this._lastMousePos=this._map.mouseEventToContainerPoint(t),this._startTime||(this._startTime=+new Date);var o=Math.max(n-(+new Date-this._startTime),0);clearTimeout(this._timer),this._timer=setTimeout(e(this._performZoom,this),o),Q(t)},_performZoom:function(){var t=this._map,i=t.getZoom(),e=this._map.options.zoomSnap||0;t._stop();var n=this._delta/(4*this._map.options.wheelPxPerZoomLevel),o=4*Math.log(2/(1+Math.exp(-Math.abs(n))))/Math.LN2,s=e?Math.ceil(o/e)*e:o,r=t._limitZoom(i+(this._delta>0?s:-s))-i;this._delta=0,this._startTime=null,r&&("center"===t.options.scrollWheelZoom?t.setZoom(i+r):t.setZoomAround(this._lastMousePos,i+r))}});Le.addInitHook("addHandler","scrollWheelZoom",zn),Le.mergeOptions({tap:!0,tapTolerance:15});var Mn=Ze.extend({addHooks:function(){V(this._map._container,"touchstart",this._onDown,this)},removeHooks:function(){q(this._map._container,"touchstart",this._onDown,this)},_onDown:function(t){if(t.touches){if($(t),this._fireClick=!0,t.touches.length>1)return this._fireClick=!1,void clearTimeout(this._holdTimeout);var i=t.touches[0],n=i.target;this._startPos=this._newPos=new x(i.clientX,i.clientY),n.tagName&&"a"===n.tagName.toLowerCase()&&pt(n,"leaflet-active"),this._holdTimeout=setTimeout(e(function(){this._isTapValid()&&(this._fireClick=!1,this._onUp(),this._simulateEvent("contextmenu",i))},this),1e3),this._simulateEvent("mousedown",i),V(document,{touchmove:this._onMove,touchend:this._onUp},this)}},_onUp:function(t){if(clearTimeout(this._holdTimeout),q(document,{touchmove:this._onMove,touchend:this._onUp},this),this._fireClick&&t&&t.changedTouches){var i=t.changedTouches[0],e=i.target;e&&e.tagName&&"a"===e.tagName.toLowerCase()&&mt(e,"leaflet-active"),this._simulateEvent("mouseup",i),this._isTapValid()&&this._simulateEvent("click",i)}},_isTapValid:function(){return this._newPos.distanceTo(this._startPos)<=this._map.options.tapTolerance},_onMove:function(t){var i=t.touches[0];this._newPos=new x(i.clientX,i.clientY),this._simulateEvent("mousemove",i)},_simulateEvent:function(t,i){var e=document.createEvent("MouseEvents");e._simulated=!0,i.target._simulatedClick=!0,e.initMouseEvent(t,!0,!0,window,1,i.screenX,i.screenY,i.clientX,i.clientY,!1,!1,!1,!1,0,null),i.target.dispatchEvent(e)}});Vi&&!Ui&&Le.addInitHook("addHandler","tap",Mn),Le.mergeOptions({touchZoom:Vi&&!zi,bounceAtZoomLimits:!0});var Cn=Ze.extend({addHooks:function(){pt(this._map._container,"leaflet-touch-zoom"),V(this._map._container,"touchstart",this._onTouchStart,this)},removeHooks:function(){mt(this._map._container,"leaflet-touch-zoom"),q(this._map._container,"touchstart",this._onTouchStart,this)},_onTouchStart:function(t){var i=this._map;if(t.touches&&2===t.touches.length&&!i._animatingZoom&&!this._zooming){var e=i.mouseEventToContainerPoint(t.touches[0]),n=i.mouseEventToContainerPoint(t.touches[1]);this._centerPoint=i.getSize()._divideBy(2),this._startLatLng=i.containerPointToLatLng(this._centerPoint),"center"!==i.options.touchZoom&&(this._pinchStartLatLng=i.containerPointToLatLng(e.add(n)._divideBy(2))),this._startDist=e.distanceTo(n),this._startZoom=i.getZoom(),this._moved=!1,this._zooming=!0,i._stop(),V(document,"touchmove",this._onTouchMove,this),V(document,"touchend",this._onTouchEnd,this),$(t)}},_onTouchMove:function(t){if(t.touches&&2===t.touches.length&&this._zooming){var i=this._map,n=i.mouseEventToContainerPoint(t.touches[0]),o=i.mouseEventToContainerPoint(t.touches[1]),s=n.distanceTo(o)/this._startDist;if(this._zoom=i.getScaleZoom(s,this._startZoom),!i.options.bounceAtZoomLimits&&(this._zoom<i.getMinZoom()&&s<1||this._zoom>i.getMaxZoom()&&s>1)&&(this._zoom=i._limitZoom(this._zoom)),"center"===i.options.touchZoom){if(this._center=this._startLatLng,1===s)return}else{var r=n._add(o)._divideBy(2)._subtract(this._centerPoint);if(1===s&&0===r.x&&0===r.y)return;this._center=i.unproject(i.project(this._pinchStartLatLng,this._zoom).subtract(r),this._zoom)}this._moved||(i._moveStart(!0,!1),this._moved=!0),g(this._animRequest);var a=e(i._move,i,this._center,this._zoom,{pinch:!0,round:!1});this._animRequest=f(a,this,!0),$(t)}},_onTouchEnd:function(){this._moved&&this._zooming?(this._zooming=!1,g(this._animRequest),q(document,"touchmove",this._onTouchMove),q(document,"touchend",this._onTouchEnd),this._map.options.zoomAnimation?this._map._animateZoom(this._center,this._map._limitZoom(this._zoom),!0,this._map.options.zoomSnap):this._map._resetView(this._center,this._map._limitZoom(this._zoom))):this._zooming=!1}});Le.addInitHook("addHandler","touchZoom",Cn),Le.BoxZoom=Ln,Le.DoubleClickZoom=Pn,Le.Drag=bn,Le.Keyboard=Tn,Le.ScrollWheelZoom=zn,Le.Tap=Mn,Le.TouchZoom=Cn;var Zn=window.L;window.L=t,Object.freeze=$t,t.version="1.3.1+HEAD.ba6f97f",t.noConflict=function(){return window.L=Zn,this},t.Control=Pe,t.control=be,t.Browser=$i,t.Evented=ui,t.Mixin=Ee,t.Util=ai,t.Class=v,t.Handler=Ze,t.extend=i,t.bind=e,t.stamp=n,t.setOptions=l,t.DomEvent=de,t.DomUtil=xe,t.PosAnimation=we,t.Draggable=Be,t.LineUtil=Oe,t.PolyUtil=Re,t.Point=x,t.point=w,t.Bounds=P,t.bounds=b,t.Transformation=Z,t.transformation=S,t.Projection=je,t.LatLng=M,t.latLng=C,t.LatLngBounds=T,t.latLngBounds=z,t.CRS=ci,t.GeoJSON=nn,t.geoJSON=Kt,t.geoJson=sn,t.Layer=Ue,t.LayerGroup=Ve,t.layerGroup=function(t,i){return new Ve(t,i)},t.FeatureGroup=qe,t.featureGroup=function(t){return new qe(t)},t.ImageOverlay=rn,t.imageOverlay=function(t,i,e){return new rn(t,i,e)},t.VideoOverlay=an,t.videoOverlay=function(t,i,e){return new an(t,i,e)},t.DivOverlay=hn,t.Popup=un,t.popup=function(t,i){return new un(t,i)},t.Tooltip=ln,t.tooltip=function(t,i){return new ln(t,i)},t.Icon=Ge,t.icon=function(t){return new Ge(t)},t.DivIcon=cn,t.divIcon=function(t){return new cn(t)},t.Marker=Xe,t.marker=function(t,i){return new Xe(t,i)},t.TileLayer=dn,t.tileLayer=Yt,t.GridLayer=_n,t.gridLayer=function(t){return new _n(t)},t.SVG=xn,t.svg=Jt,t.Renderer=mn,t.Canvas=fn,t.canvas=Xt,t.Path=Je,t.CircleMarker=$e,t.circleMarker=function(t,i){return new $e(t,i)},t.Circle=Qe,t.circle=function(t,i,e){return new Qe(t,i,e)},t.Polyline=tn,t.polyline=function(t,i){return new tn(t,i)},t.Polygon=en,t.polygon=function(t,i){return new en(t,i)},t.Rectangle=wn,t.rectangle=function(t,i){return new wn(t,i)},t.Map=Le,t.map=function(t,i){return new Le(t,i)}});</script>
<style type="text/css">
img.leaflet-tile {
padding: 0;
margin: 0;
border-radius: 0;
border: none;
}
.leaflet .info {
padding: 6px 8px;
font: 14px/16px Arial, Helvetica, sans-serif;
background: white;
background: rgba(255,255,255,0.8);
box-shadow: 0 0 15px rgba(0,0,0,0.2);
border-radius: 5px;
}
.leaflet .legend {
line-height: 18px;
color: #555;
}
.leaflet .legend svg text {
fill: #555;
}
.leaflet .legend svg line {
stroke: #555;
}
.leaflet .legend i {
width: 18px;
height: 18px;
margin-right: 4px;
opacity: 0.7;
display: inline-block;
vertical-align: top;

zoom: 1;
*display: inline;
}
</style>
<script>!function(t,s){"object"==typeof exports&&"undefined"!=typeof module?module.exports=s():"function"==typeof define&&define.amd?define(s):t.proj4=s()}(this,function(){"use strict";function k(t,s){if(t[s])return t[s];for(var i,a=Object.keys(t),h=s.toLowerCase().replace(H,""),e=-1;++e<a.length;)if((i=a[e]).toLowerCase().replace(H,"")===h)return t[i]}function e(t){if("string"!=typeof t)throw new Error("not a string");this.text=t.trim(),this.level=0,this.place=0,this.root=null,this.stack=[],this.currentObject=null,this.state=K}function h(t,s,i){Array.isArray(s)&&(i.unshift(s),s=null);var a=s?{}:t,h=i.reduce(function(t,s){return n(s,t),t},a);s&&(t[s]=h)}function n(t,s){if(Array.isArray(t)){var i,a=t.shift();if("PARAMETER"===a&&(a=t.shift()),1===t.length)return Array.isArray(t[0])?(s[a]={},void n(t[0],s[a])):void(s[a]=t[0]);if(t.length)if("TOWGS84"!==a){if("AXIS"===a)return a in s||(s[a]=[]),void s[a].push(t);switch(Array.isArray(a)||(s[a]={}),a){case"UNIT":case"PRIMEM":case"VERT_DATUM":return s[a]={name:t[0].toLowerCase(),convert:t[1]},void(3===t.length&&n(t[2],s[a]));case"SPHEROID":case"ELLIPSOID":return s[a]={name:t[0],a:t[1],rf:t[2]},void(4===t.length&&n(t[3],s[a]));case"PROJECTEDCRS":case"PROJCRS":case"GEOGCS":case"GEOCCS":case"PROJCS":case"LOCAL_CS":case"GEODCRS":case"GEODETICCRS":case"GEODETICDATUM":case"EDATUM":case"ENGINEERINGDATUM":case"VERT_CS":case"VERTCRS":case"VERTICALCRS":case"COMPD_CS":case"COMPOUNDCRS":case"ENGINEERINGCRS":case"ENGCRS":case"FITTED_CS":case"LOCAL_DATUM":case"DATUM":return t[0]=["name",t[0]],void h(s,a,t);default:for(i=-1;++i<t.length;)if(!Array.isArray(t[i]))return n(t,s[a]);return h(s,a,t)}}else s[a]=t;else s[a]=!0}else s[t]=!0}function r(t){return t*it}function o(e){function t(t){return t*(e.to_meter||1)}if("GEOGCS"===e.type?e.projName="longlat":"LOCAL_CS"===e.type?(e.projName="identity",e.local=!0):"object"==typeof e.PROJECTION?e.projName=Object.keys(e.PROJECTION)[0]:e.projName=e.PROJECTION,e.AXIS){for(var s="",i=0,a=e.AXIS.length;i<a;++i){var h=e.AXIS[i][0].toLowerCase();-1!==h.indexOf("north")?s+="n":-1!==h.indexOf("south")?s+="s":-1!==h.indexOf("east")?s+="e":-1!==h.indexOf("west")&&(s+="w")}2===s.length&&(s+="u"),3===s.length&&(e.axis=s)}e.UNIT&&(e.units=e.UNIT.name.toLowerCase(),"metre"===e.units&&(e.units="meter"),e.UNIT.convert&&("GEOGCS"===e.type?e.DATUM&&e.DATUM.SPHEROID&&(e.to_meter=e.UNIT.convert*e.DATUM.SPHEROID.a):e.to_meter=e.UNIT.convert));var n=e.GEOGCS;"GEOGCS"===e.type&&(n=e),n&&(n.DATUM?e.datumCode=n.DATUM.name.toLowerCase():e.datumCode=n.name.toLowerCase(),"d_"===e.datumCode.slice(0,2)&&(e.datumCode=e.datumCode.slice(2)),"new_zealand_geodetic_datum_1949"!==e.datumCode&&"new_zealand_1949"!==e.datumCode||(e.datumCode="nzgd49"),"wgs_1984"!==e.datumCode&&"world_geodetic_system_1984"!==e.datumCode||("Mercator_Auxiliary_Sphere"===e.PROJECTION&&(e.sphere=!0),e.datumCode="wgs84"),"_ferro"===e.datumCode.slice(-6)&&(e.datumCode=e.datumCode.slice(0,-6)),"_jakarta"===e.datumCode.slice(-8)&&(e.datumCode=e.datumCode.slice(0,-8)),~e.datumCode.indexOf("belge")&&(e.datumCode="rnb72"),n.DATUM&&n.DATUM.SPHEROID&&(e.ellps=n.DATUM.SPHEROID.name.replace("_19","").replace(/[Cc]larke\_18/,"clrk"),"international"===e.ellps.toLowerCase().slice(0,13)&&(e.ellps="intl"),e.a=n.DATUM.SPHEROID.a,e.rf=parseFloat(n.DATUM.SPHEROID.rf,10)),n.DATUM&&n.DATUM.TOWGS84&&(e.datum_params=n.DATUM.TOWGS84),~e.datumCode.indexOf("osgb_1936")&&(e.datumCode="osgb36"),~e.datumCode.indexOf("osni_1952")&&(e.datumCode="osni52"),(~e.datumCode.indexOf("tm65")||~e.datumCode.indexOf("geodetic_datum_of_1965"))&&(e.datumCode="ire65"),"ch1903+"===e.datumCode&&(e.datumCode="ch1903"),~e.datumCode.indexOf("israel")&&(e.datumCode="isr93")),e.b&&!isFinite(e.b)&&(e.b=e.a),[["standard_parallel_1","Standard_Parallel_1"],["standard_parallel_2","Standard_Parallel_2"],["false_easting","False_Easting"],["false_northing","False_Northing"],["central_meridian","Central_Meridian"],["latitude_of_origin","Latitude_Of_Origin"],["latitude_of_origin","Central_Parallel"],["scale_factor","Scale_Factor"],["k0","scale_factor"],["latitude_of_center","Latitude_Of_Center"],["latitude_of_center","Latitude_of_center"],["lat0","latitude_of_center",r],["longitude_of_center","Longitude_Of_Center"],["longitude_of_center","Longitude_of_center"],["longc","longitude_of_center",r],["x0","false_easting",t],["y0","false_northing",t],["long0","central_meridian",r],["lat0","latitude_of_origin",r],["lat0","standard_parallel_1",r],["lat1","standard_parallel_1",r],["lat2","standard_parallel_2",r],["azimuth","Azimuth"],["alpha","azimuth",r],["srsCode","name"]].forEach(function(t){return s=e,a=(i=t)[0],h=i[1],void(!(a in s)&&h in s&&(s[a]=s[h],3===i.length&&(s[a]=i[2](s[a]))));var s,i,a,h}),e.long0||!e.longc||"Albers_Conic_Equal_Area"!==e.projName&&"Lambert_Azimuthal_Equal_Area"!==e.projName||(e.long0=e.longc),e.lat_ts||!e.lat1||"Stereographic_South_Pole"!==e.projName&&"Polar Stereographic (variant B)"!==e.projName||(e.lat0=r(0<e.lat1?90:-90),e.lat_ts=e.lat1)}function l(t){var s=this;if(2===arguments.length){var i=arguments[1];"string"==typeof i?"+"===i.charAt(0)?l[t]=J(arguments[1]):l[t]=at(arguments[1]):l[t]=i}else if(1===arguments.length){if(Array.isArray(t))return t.map(function(t){Array.isArray(t)?l.apply(s,t):l(t)});if("string"==typeof t){if(t in l)return l[t]}else"EPSG"in t?l["EPSG:"+t.EPSG]=t:"ESRI"in t?l["ESRI:"+t.ESRI]=t:"IAU2000"in t?l["IAU2000:"+t.IAU2000]=t:console.log(t);return}}function E(t){if("string"!=typeof t)return t;if(t in l)return l[t];if(a=t,lt.some(function(t){return-1<a.indexOf(t)})){var s=at(t);if(function(t){var s=k(t,"authority");if(s){var i=k(s,"epsg");return i&&-1<Mt.indexOf(i)}}(s))return l["EPSG:3857"];var i=function(t){var s=k(t,"extension");if(s)return k(s,"proj4")}(s);return i?J(i):s}var a;return"+"===t[0]?J(t):void 0}function t(t){return t}function s(t,s){var i=mt.length;return t.names?((mt[i]=t).names.forEach(function(t){ft[t.toLowerCase()]=i}),this):(console.log(s),!0)}function q(t,s){if(!(this instanceof q))return new q(t);s=s||function(t){if(t)throw t};var i,a,h,e,n,r,o,l,M,c,u,f,m,p,d,y,_,x,g,b,v,w,C,P,S,N=E(t);"object"==typeof N&&(i=q.projections.get(N.projName))?(!N.datumCode||"none"===N.datumCode||(a=k(_t,N.datumCode))&&(N.datum_params=a.towgs84?a.towgs84.split(","):null,N.ellps=a.ellipse,N.datumName=a.datumName?a.datumName:N.datumCode),N.k0=N.k0||1,N.axis=N.axis||"enu",N.ellps=N.ellps||"wgs84",b=N.a,v=N.b,w=N.rf,C=N.ellps,P=N.sphere,b||(b=(S=(S=k(dt,C))||yt).a,v=S.b,w=S.rf),w&&!v&&(v=(1-1/w)*b),(0===w||Math.abs(b-v)<D)&&(P=!0,v=b),m=(h={a:b,b:v,rf:w,sphere:P}).a,p=h.b,d=N.R_A,x=((y=m*m)-(_=p*p))/y,g=0,d?(y=(m*=1-x*(R+x*(L+x*T)))*m,x=0):g=Math.sqrt(x),e={es:x,e:g,ep2:(y-_)/_},n=N.datum||(r=N.datumCode,o=N.datum_params,l=h.a,M=h.b,c=e.es,u=e.ep2,(f={}).datum_type=void 0===r||"none"===r?G:A,o&&(f.datum_params=o.map(parseFloat),0===f.datum_params[0]&&0===f.datum_params[1]&&0===f.datum_params[2]||(f.datum_type=I),3<f.datum_params.length&&(0===f.datum_params[3]&&0===f.datum_params[4]&&0===f.datum_params[5]&&0===f.datum_params[6]||(f.datum_type=O,f.datum_params[3]*=j,f.datum_params[4]*=j,f.datum_params[5]*=j,f.datum_params[6]=f.datum_params[6]/1e6+1))),f.a=l,f.b=M,f.es=c,f.ep2=u,f),ct(this,N),ct(this,i),this.a=h.a,this.b=h.b,this.rf=h.rf,this.sphere=h.sphere,this.es=e.es,this.e=e.e,this.ep2=e.ep2,this.datum=n,this.init(),s(null,this)):s(t)}function M(t,s,i){var a,h,e,n,r=t.x,o=t.y,l=t.z?t.z:0;if(o<-z&&-1.001*z<o)o=-z;else if(z<o&&o<1.001*z)o=z;else{if(o<-z)return{x:-1/0,y:-1/0,z:t.z};if(z<o)return{x:1/0,y:1/0,z:t.z}}return r>Math.PI&&(r-=2*Math.PI),h=Math.sin(o),n=Math.cos(o),e=h*h,{x:((a=i/Math.sqrt(1-s*e))+l)*n*Math.cos(r),y:(a+l)*n*Math.sin(r),z:(a*(1-s)+l)*h}}function c(t,s,i,a){var h,e,n,r,o,l,M,c,u,f,m,p,d,y=t.x,_=t.y,x=t.z?t.z:0,g=Math.sqrt(y*y+_*_),b=Math.sqrt(y*y+_*_+x*x);if(g/i<1e-12){if(p=0,b/i<1e-12)return d=-a,{x:t.x,y:t.y,z:t.z}}else p=Math.atan2(_,y);for(h=x/b,l=(e=g/b)*(1-s)*(n=1/Math.sqrt(1-s*(2-s)*e*e)),M=h*n,m=0;m++,r=s*(o=i/Math.sqrt(1-s*M*M))/(o+(d=g*l+x*M-o*(1-s*M*M))),f=(u=h*(n=1/Math.sqrt(1-r*(2-r)*e*e)))*l-(c=e*(1-r)*n)*M,l=c,M=u,1e-24<f*f&&m<30;);return{x:p,y:Math.atan(u/Math.abs(c)),z:d}}function u(t){return t===I||t===O}function i(t){if("function"==typeof Number.isFinite){if(Number.isFinite(t))return;throw new TypeError("coordinates must be finite numbers")}if("number"!=typeof t||t!=t||!isFinite(t))throw new TypeError("coordinates must be finite numbers")}function f(t,s,i){var a,h,e;if(Array.isArray(i)&&(i=bt(i)),vt(i),t.datum&&s.datum&&(e=s,((h=t).datum.datum_type===I||h.datum.datum_type===O)&&"WGS84"!==e.datumCode||(e.datum.datum_type===I||e.datum.datum_type===O)&&"WGS84"!==h.datumCode)&&(i=f(t,a=new q("WGS84"),i),t=a),"enu"!==t.axis&&(i=gt(t,!1,i)),"longlat"===t.projName)i={x:i.x*N,y:i.y*N,z:i.z||0};else if(t.to_meter&&(i={x:i.x*t.to_meter,y:i.y*t.to_meter,z:i.z||0}),!(i=t.inverse(i)))return;return t.from_greenwich&&(i.x+=t.from_greenwich),i=xt(t.datum,s.datum,i),s.from_greenwich&&(i={x:i.x-s.from_greenwich,y:i.y,z:i.z||0}),"longlat"===s.projName?i={x:i.x*B,y:i.y*B,z:i.z||0}:(i=s.forward(i),s.to_meter&&(i={x:i.x/s.to_meter,y:i.y/s.to_meter,z:i.z||0})),"enu"!==s.axis?gt(s,!0,i):i}function m(s,i,a){var t,h,e;return Array.isArray(a)?(t=f(s,i,a)||{x:NaN,y:NaN},2<a.length?void 0!==s.name&&"geocent"===s.name||void 0!==i.name&&"geocent"===i.name?"number"==typeof t.z?[t.x,t.y,t.z].concat(a.splice(3)):[t.x,t.y,a[2]].concat(a.splice(3)):[t.x,t.y].concat(a.splice(2)):[t.x,t.y]):(h=f(s,i,a),2===(e=Object.keys(a)).length||e.forEach(function(t){if(void 0!==s.name&&"geocent"===s.name||void 0!==i.name&&"geocent"===i.name){if("x"===t||"y"===t||"z"===t)return}else if("x"===t||"y"===t)return;h[t]=a[t]}),h)}function p(t){return t instanceof q?t:t.oProj?t.oProj:q(t)}function a(s,i,t){s=p(s);var a,h=!1;return void 0===i?(i=s,s=wt,h=!0):void 0===i.x&&!Array.isArray(i)||(t=i,i=s,s=wt,h=!0),i=p(i),t?m(s,i,t):(a={forward:function(t){return m(s,i,t)},inverse:function(t){return m(i,s,t)}},h&&(a.oProj=i),a)}function d(t,s){return s=s||5,i=function(t){var s,i,a,h,e,n,r=t.lat,o=t.lon,l=_(r),M=_(o);n=Math.floor((o+180)/6)+1,180===o&&(n=60),56<=r&&r<64&&3<=o&&o<12&&(n=32),72<=r&&r<84&&(0<=o&&o<9?n=31:9<=o&&o<21?n=33:21<=o&&o<33?n=35:33<=o&&o<42&&(n=37)),e=_(6*(n-1)-180+3),s=6378137/Math.sqrt(1-.00669438*Math.sin(l)*Math.sin(l)),i=Math.tan(l)*Math.tan(l),a=.006739496752268451*Math.cos(l)*Math.cos(l);var c=.9996*s*((h=Math.cos(l)*(M-e))+(1-i+a)*h*h*h/6+(5-18*i+i*i+72*a-.39089081163157013)*h*h*h*h*h/120)+5e5,u=.9996*(6378137*(.9983242984503243*l-.002514607064228144*Math.sin(2*l)+2639046602129982e-21*Math.sin(4*l)-3.418046101696858e-9*Math.sin(6*l))+s*Math.tan(l)*(h*h/2+(5-i+9*a+4*a*a)*h*h*h*h/24+(61-58*i+i*i+600*a-2.2240339282485886)*h*h*h*h*h*h/720));return r<0&&(u+=1e7),{northing:Math.round(u),easting:Math.round(c),zoneNumber:n,zoneLetter:function(t){var s="Z";return t<=84&&72<=t?s="X":t<72&&64<=t?s="W":t<64&&56<=t?s="V":t<56&&48<=t?s="U":t<48&&40<=t?s="T":t<40&&32<=t?s="S":t<32&&24<=t?s="R":t<24&&16<=t?s="Q":t<16&&8<=t?s="P":t<8&&0<=t?s="N":t<0&&-8<=t?s="M":t<-8&&-16<=t?s="L":t<-16&&-24<=t?s="K":t<-24&&-32<=t?s="J":t<-32&&-40<=t?s="H":t<-40&&-48<=t?s="G":t<-48&&-56<=t?s="F":t<-56&&-64<=t?s="E":t<-64&&-72<=t?s="D":t<-72&&-80<=t&&(s="C"),s}(r)}}({lat:t[1],lon:t[0]}),a=s,h="00000"+i.easting,e="00000"+i.northing,i.zoneNumber+i.zoneLetter+function(t,s,i){var a=b(i);return function(t,s,i){var a=i-1,h=Pt.charCodeAt(a),e=St.charCodeAt(a),n=h+t-1,r=e+s,o=!1;return It<n&&(n=n-It+Nt-1,o=!0),(n===kt||h<kt&&kt<n||(kt<n||h<kt)&&o)&&n++,(n===Et||h<Et&&Et<n||(Et<n||h<Et)&&o)&&++n===kt&&n++,It<n&&(n=n-It+Nt-1),o=qt<r&&(r=r-qt+Nt-1,!0),(r===kt||e<kt&&kt<r||(kt<r||e<kt)&&o)&&r++,(r===Et||e<Et&&Et<r||(Et<r||e<Et)&&o)&&++r===kt&&r++,qt<r&&(r=r-qt+Nt-1),String.fromCharCode(n)+String.fromCharCode(r)}(Math.floor(t/1e5),Math.floor(s/1e5)%20,a)}(i.easting,i.northing,i.zoneNumber)+h.substr(h.length-5,a)+e.substr(e.length-5,a);var i,a,h,e}function y(t){var s=g(v(t.toUpperCase()));return s.lat&&s.lon?[s.lon,s.lat]:[(s.left+s.right)/2,(s.top+s.bottom)/2]}function _(t){return t*(Math.PI/180)}function x(t){return t/Math.PI*180}function g(t){var s=t.northing,i=t.easting,a=t.zoneLetter,h=t.zoneNumber;if(h<0||60<h)return null;var e,n,r,o,l,M,c,u,f=(1-Math.sqrt(.99330562))/(1+Math.sqrt(.99330562)),m=i-5e5,p=s;a<"N"&&(p-=1e7),M=6*(h-1)-180+3,u=(c=p/.9996/6367449.145945056)+(3*f/2-27*f*f*f/32)*Math.sin(2*c)+(21*f*f/16-55*f*f*f*f/32)*Math.sin(4*c)+151*f*f*f/96*Math.sin(6*c),e=6378137/Math.sqrt(1-.00669438*Math.sin(u)*Math.sin(u)),n=Math.tan(u)*Math.tan(u),r=.006739496752268451*Math.cos(u)*Math.cos(u),o=6335439.32722994/Math.pow(1-.00669438*Math.sin(u)*Math.sin(u),1.5),l=m/(.9996*e);var d,y=x(y=u-e*Math.tan(u)/o*(l*l/2-(5+3*n+10*r-4*r*r-.06065547077041606)*l*l*l*l/24+(61+90*n+298*r+45*n*n-1.6983531815716497-3*r*r)*l*l*l*l*l*l/720)),_=M+x(_=(l-(1+2*n+r)*l*l*l/6+(5-2*r+28*n-3*r*r+.05391597401814761+24*n*n)*l*l*l*l*l/120)/Math.cos(u));return t.accuracy?{top:(d=g({northing:t.northing+t.accuracy,easting:t.easting+t.accuracy,zoneLetter:t.zoneLetter,zoneNumber:t.zoneNumber})).lat,right:d.lon,bottom:y,left:_}:{lat:y,lon:_}}function b(t){var s=t%Ct;return 0===s&&(s=Ct),s}function v(t){if(t&&0===t.length)throw"MGRSPoint coverting from nothing";for(var s,i=t.length,a=null,h="",e=0;!/[A-Z]/.test(s=t.charAt(e));){if(2<=e)throw"MGRSPoint bad conversion from: "+t;h+=s,e++}var n=parseInt(h,10);if(0===e||i<e+3)throw"MGRSPoint bad conversion from: "+t;var r=t.charAt(e++);if(r<="A"||"B"===r||"Y"===r||"Z"<=r||"I"===r||"O"===r)throw"MGRSPoint zone letter "+r+" not handled: "+t;a=t.substring(e,e+=2);for(var o=b(n),l=function(t,s){for(var i=Pt.charCodeAt(s-1),a=1e5,h=!1;i!==t.charCodeAt(0);){if(++i===kt&&i++,i===Et&&i++,It<i){if(h)throw"Bad character: "+t;i=Nt,h=!0}a+=1e5}return a}(a.charAt(0),o),M=function(t,s){if("V"<t)throw"MGRSPoint given invalid Northing "+t;for(var i=St.charCodeAt(s-1),a=0,h=!1;i!==t.charCodeAt(0);){if(++i===kt&&i++,i===Et&&i++,qt<i){if(h)throw"Bad character: "+t;i=Nt,h=!0}a+=1e5}return a}(a.charAt(1),o);M<w(r);)M+=2e6;var c=i-e;if(c%2!=0)throw"MGRSPoint has to have an even number \nof digits after the zone letter and two 100km letters - front \nhalf for easting meters, second half for \nnorthing meters"+t;var u,f,m,p=c/2,d=0,y=0;return 0<p&&(u=1e5/Math.pow(10,p),f=t.substring(e,e+p),d=parseFloat(f)*u,m=t.substring(e+p),y=parseFloat(m)*u),{easting:d+l,northing:y+M,zoneLetter:r,zoneNumber:n,accuracy:u}}function w(t){var s;switch(t){case"C":s=11e5;break;case"D":s=2e6;break;case"E":s=28e5;break;case"F":s=37e5;break;case"G":s=46e5;break;case"H":s=55e5;break;case"J":s=64e5;break;case"K":s=73e5;break;case"L":s=82e5;break;case"M":s=91e5;break;case"N":s=0;break;case"P":s=8e5;break;case"Q":s=17e5;break;case"R":s=26e5;break;case"S":s=35e5;break;case"T":s=44e5;break;case"U":s=53e5;break;case"V":s=62e5;break;case"W":s=7e6;break;case"X":s=79e5;break;default:s=-1}if(0<=s)return s;throw"Invalid zone letter: "+t}function C(t,s,i){if(!(this instanceof C))return new C(t,s,i);var a;Array.isArray(t)?(this.x=t[0],this.y=t[1],this.z=t[2]||0):"object"==typeof t?(this.x=t.x,this.y=t.y,this.z=t.z||0):"string"==typeof t&&void 0===s?(a=t.split(","),this.x=parseFloat(a[0],10),this.y=parseFloat(a[1],10),this.z=parseFloat(a[2],10)||0):(this.x=t,this.y=s,this.z=i||0),console.warn("proj4.Point will be removed in version 3, use proj4.toPoint")}function P(t,s,i,a){var h;return t<D?(a.value=Os,h=0):(h=Math.atan2(s,i),Math.abs(h)<=U?a.value=Os:U<h&&h<=z+U?(a.value=As,h-=z):z+U<h||h<=-(z+U)?(a.value=Gs,h=0<=h?h-Q:h+Q):(a.value=js,h+=z)),h}function S(t,s){var i=t+s;return i<-Q?i+=F:+Q<i&&(i-=F),i}var I=1,O=2,A=4,G=5,j=484813681109536e-20,z=Math.PI/2,R=.16666666666666666,L=.04722222222222222,T=.022156084656084655,D=1e-10,N=.017453292519943295,B=57.29577951308232,U=Math.PI/4,F=2*Math.PI,Q=3.14159265359,W={greenwich:0,lisbon:-9.131906111111,paris:2.337229166667,bogota:-74.080916666667,madrid:-3.687938888889,rome:12.452333333333,bern:7.439583333333,jakarta:106.807719444444,ferro:-17.666666666667,brussels:4.367975,stockholm:18.058277777778,athens:23.7163375,oslo:10.722916666667},X={ft:{to_meter:.3048},"us-ft":{to_meter:1200/3937}},H=/[\s_\-\/\(\)]/g,J=function(t){var s,i,a,h={},e=t.split("+").map(function(t){return t.trim()}).filter(function(t){return t}).reduce(function(t,s){var i=s.split("=");return i.push(!0),t[i[0].toLowerCase()]=i[1],t},{}),n={proj:"projName",datum:"datumCode",rf:function(t){h.rf=parseFloat(t)},lat_0:function(t){h.lat0=t*N},lat_1:function(t){h.lat1=t*N},lat_2:function(t){h.lat2=t*N},lat_ts:function(t){h.lat_ts=t*N},lon_0:function(t){h.long0=t*N},lon_1:function(t){h.long1=t*N},lon_2:function(t){h.long2=t*N},alpha:function(t){h.alpha=parseFloat(t)*N},lonc:function(t){h.longc=t*N},x_0:function(t){h.x0=parseFloat(t)},y_0:function(t){h.y0=parseFloat(t)},k_0:function(t){h.k0=parseFloat(t)},k:function(t){h.k0=parseFloat(t)},a:function(t){h.a=parseFloat(t)},b:function(t){h.b=parseFloat(t)},r_a:function(){h.R_A=!0},zone:function(t){h.zone=parseInt(t,10)},south:function(){h.utmSouth=!0},towgs84:function(t){h.datum_params=t.split(",").map(function(t){return parseFloat(t)})},to_meter:function(t){h.to_meter=parseFloat(t)},units:function(t){h.units=t;var s=k(X,t);s&&(h.to_meter=s.to_meter)},from_greenwich:function(t){h.from_greenwich=t*N},pm:function(t){var s=k(W,t);h.from_greenwich=(s||parseFloat(t))*N},nadgrids:function(t){"@null"===t?h.datumCode="none":h.nadgrids=t},axis:function(t){3===t.length&&-1!=="ewnsud".indexOf(t.substr(0,1))&&-1!=="ewnsud".indexOf(t.substr(1,1))&&-1!=="ewnsud".indexOf(t.substr(2,1))&&(h.axis=t)}};for(s in e)i=e[s],s in n?"function"==typeof(a=n[s])?a(i):h[a]=i:h[s]=i;return"string"==typeof h.datumCode&&"WGS84"!==h.datumCode&&(h.datumCode=h.datumCode.toLowerCase()),h},K=1,V=/\s/,Z=/[A-Za-z]/,Y=/[A-Za-z84]/,$=/[,\]]/,tt=/[\d\.E\-\+]/;e.prototype.readCharicter=function(){var t=this.text[this.place++];if(4!==this.state)for(;V.test(t);){if(this.place>=this.text.length)return;t=this.text[this.place++]}switch(this.state){case K:return this.neutral(t);case 2:return this.keyword(t);case 4:return this.quoted(t);case 5:return this.afterquote(t);case 3:return this.number(t);case-1:return}},e.prototype.afterquote=function(t){if('"'===t)return this.word+='"',void(this.state=4);if($.test(t))return this.word=this.word.trim(),void this.afterItem(t);throw new Error("havn't handled \""+t+'" in afterquote yet, index '+this.place)},e.prototype.afterItem=function(t){return","===t?(null!==this.word&&this.currentObject.push(this.word),this.word=null,void(this.state=K)):"]"===t?(this.level--,null!==this.word&&(this.currentObject.push(this.word),this.word=null),this.state=K,this.currentObject=this.stack.pop(),void(this.currentObject||(this.state=-1))):void 0},e.prototype.number=function(t){if(!tt.test(t)){if($.test(t))return this.word=parseFloat(this.word),void this.afterItem(t);throw new Error("havn't handled \""+t+'" in number yet, index '+this.place)}this.word+=t},e.prototype.quoted=function(t){'"'!==t?this.word+=t:this.state=5},e.prototype.keyword=function(t){if(Y.test(t))this.word+=t;else{if("["===t){var s=[];return s.push(this.word),this.level++,null===this.root?this.root=s:this.currentObject.push(s),this.stack.push(this.currentObject),this.currentObject=s,void(this.state=K)}if(!$.test(t))throw new Error("havn't handled \""+t+'" in keyword yet, index '+this.place);this.afterItem(t)}},e.prototype.neutral=function(t){if(Z.test(t))return this.word=t,void(this.state=2);if('"'===t)return this.word="",void(this.state=4);if(tt.test(t))return this.word=t,void(this.state=3);if(!$.test(t))throw new Error("havn't handled \""+t+'" in neutral yet, index '+this.place);this.afterItem(t)},e.prototype.output=function(){for(;this.place<this.text.length;)this.readCharicter();if(-1===this.state)return this.root;throw new Error('unable to parse string "'+this.text+'". State is '+this.state)};var st,it=.017453292519943295,at=function(t){var s=new e(t).output(),i=s.shift(),a=s.shift();s.unshift(["name",a]),s.unshift(["type",i]);var h={};return n(s,h),o(h),h};(st=l)("EPSG:4326","+title=WGS 84 (long/lat) +proj=longlat +ellps=WGS84 +datum=WGS84 +units=degrees"),st("EPSG:4269","+title=NAD83 (long/lat) +proj=longlat +a=6378137.0 +b=6356752.31414036 +ellps=GRS80 +datum=NAD83 +units=degrees"),st("EPSG:3857","+title=WGS 84 / Pseudo-Mercator +proj=merc +a=6378137 +b=6378137 +lat_ts=0.0 +lon_0=0.0 +x_0=0.0 +y_0=0 +k=1.0 +units=m +nadgrids=@null +no_defs"),st.WGS84=st["EPSG:4326"],st["EPSG:3785"]=st["EPSG:3857"],st.GOOGLE=st["EPSG:3857"],st["EPSG:900913"]=st["EPSG:3857"],st["EPSG:102113"]=st["EPSG:3857"];function ht(t,s,i){var a=t*s;return i/Math.sqrt(1-a*a)}function et(t){return t<0?-1:1}function nt(t){return Math.abs(t)<=Q?t:t-et(t)*F}function rt(t,s,i){var a=t*i,h=.5*t,a=Math.pow((1-a)/(1+a),h);return Math.tan(.5*(z-s))/a}function ot(t,s){for(var i,a,h=.5*t,e=z-2*Math.atan(s),n=0;n<=15;n++)if(i=t*Math.sin(e),e+=a=z-2*Math.atan(s*Math.pow((1-i)/(1+i),h))-e,Math.abs(a)<=1e-10)return e;return-9999}var lt=["PROJECTEDCRS","PROJCRS","GEOGCS","GEOCCS","PROJCS","LOCAL_CS","GEODCRS","GEODETICCRS","GEODETICDATUM","ENGCRS","ENGINEERINGCRS"],Mt=["3857","900913","3785","102113"],ct=function(t,s){var i,a;if(t=t||{},!s)return t;for(a in s)void 0!==(i=s[a])&&(t[a]=i);return t},ut=[{init:function(){var t=this.b/this.a;this.es=1-t*t,"x0"in this||(this.x0=0),"y0"in this||(this.y0=0),this.e=Math.sqrt(this.es),this.lat_ts?this.sphere?this.k0=Math.cos(this.lat_ts):this.k0=ht(this.e,Math.sin(this.lat_ts),Math.cos(this.lat_ts)):this.k0||(this.k?this.k0=this.k:this.k0=1)},forward:function(t){var s,i,a,h,e=t.x,n=t.y;return 90<n*B&&n*B<-90&&180<e*B&&e*B<-180||Math.abs(Math.abs(n)-z)<=D?null:(h=this.sphere?(a=this.x0+this.a*this.k0*nt(e-this.long0),this.y0+this.a*this.k0*Math.log(Math.tan(U+.5*n))):(s=Math.sin(n),i=rt(this.e,n,s),a=this.x0+this.a*this.k0*nt(e-this.long0),this.y0-this.a*this.k0*Math.log(i)),t.x=a,t.y=h,t)},inverse:function(t){var s,i,a=t.x-this.x0,h=t.y-this.y0;if(this.sphere)i=z-2*Math.atan(Math.exp(-h/(this.a*this.k0)));else{var e=Math.exp(-h/(this.a*this.k0));if(-9999===(i=ot(this.e,e)))return null}return s=nt(this.long0+a/(this.a*this.k0)),t.x=s,t.y=i,t},names:["Mercator","Popular Visualisation Pseudo Mercator","Mercator_1SP","Mercator_Auxiliary_Sphere","merc"]},{init:function(){},forward:t,inverse:t,names:["longlat","identity"]}],ft={},mt=[],pt={start:function(){ut.forEach(s)},add:s,get:function(t){if(!t)return!1;var s=t.toLowerCase();return void 0!==ft[s]&&mt[ft[s]]?mt[ft[s]]:void 0}},dt={MERIT:{a:6378137,rf:298.257,ellipseName:"MERIT 1983"},SGS85:{a:6378136,rf:298.257,ellipseName:"Soviet Geodetic System 85"},GRS80:{a:6378137,rf:298.257222101,ellipseName:"GRS 1980(IUGG, 1980)"},IAU76:{a:6378140,rf:298.257,ellipseName:"IAU 1976"},airy:{a:6377563.396,b:6356256.91,ellipseName:"Airy 1830"},APL4:{a:6378137,rf:298.25,ellipseName:"Appl. Physics. 1965"},NWL9D:{a:6378145,rf:298.25,ellipseName:"Naval Weapons Lab., 1965"},mod_airy:{a:6377340.189,b:6356034.446,ellipseName:"Modified Airy"},andrae:{a:6377104.43,rf:300,ellipseName:"Andrae 1876 (Den., Iclnd.)"},aust_SA:{a:6378160,rf:298.25,ellipseName:"Australian Natl & S. Amer. 1969"},GRS67:{a:6378160,rf:298.247167427,ellipseName:"GRS 67(IUGG 1967)"},bessel:{a:6377397.155,rf:299.1528128,ellipseName:"Bessel 1841"},bess_nam:{a:6377483.865,rf:299.1528128,ellipseName:"Bessel 1841 (Namibia)"},clrk66:{a:6378206.4,b:6356583.8,ellipseName:"Clarke 1866"},clrk80:{a:6378249.145,rf:293.4663,ellipseName:"Clarke 1880 mod."},clrk58:{a:6378293.645208759,rf:294.2606763692654,ellipseName:"Clarke 1858"},CPM:{a:6375738.7,rf:334.29,ellipseName:"Comm. des Poids et Mesures 1799"},delmbr:{a:6376428,rf:311.5,ellipseName:"Delambre 1810 (Belgium)"},engelis:{a:6378136.05,rf:298.2566,ellipseName:"Engelis 1985"},evrst30:{a:6377276.345,rf:300.8017,ellipseName:"Everest 1830"},evrst48:{a:6377304.063,rf:300.8017,ellipseName:"Everest 1948"},evrst56:{a:6377301.243,rf:300.8017,ellipseName:"Everest 1956"},evrst69:{a:6377295.664,rf:300.8017,ellipseName:"Everest 1969"},evrstSS:{a:6377298.556,rf:300.8017,ellipseName:"Everest (Sabah & Sarawak)"},fschr60:{a:6378166,rf:298.3,ellipseName:"Fischer (Mercury Datum) 1960"},fschr60m:{a:6378155,rf:298.3,ellipseName:"Fischer 1960"},fschr68:{a:6378150,rf:298.3,ellipseName:"Fischer 1968"},helmert:{a:6378200,rf:298.3,ellipseName:"Helmert 1906"},hough:{a:6378270,rf:297,ellipseName:"Hough"},intl:{a:6378388,rf:297,ellipseName:"International 1909 (Hayford)"},kaula:{a:6378163,rf:298.24,ellipseName:"Kaula 1961"},lerch:{a:6378139,rf:298.257,ellipseName:"Lerch 1979"},mprts:{a:6397300,rf:191,ellipseName:"Maupertius 1738"},new_intl:{a:6378157.5,b:6356772.2,ellipseName:"New International 1967"},plessis:{a:6376523,rf:6355863,ellipseName:"Plessis 1817 (France)"},krass:{a:6378245,rf:298.3,ellipseName:"Krassovsky, 1942"},SEasia:{a:6378155,b:6356773.3205,ellipseName:"Southeast Asia"},walbeck:{a:6376896,b:6355834.8467,ellipseName:"Walbeck"},WGS60:{a:6378165,rf:298.3,ellipseName:"WGS 60"},WGS66:{a:6378145,rf:298.25,ellipseName:"WGS 66"},WGS7:{a:6378135,rf:298.26,ellipseName:"WGS 72"}},yt=dt.WGS84={a:6378137,rf:298.257223563,ellipseName:"WGS 84"};dt.sphere={a:6370997,b:6370997,ellipseName:"Normal Sphere (r=6370997)"};var _t={wgs84:{towgs84:"0,0,0",ellipse:"WGS84",datumName:"WGS84"},ch1903:{towgs84:"674.374,15.056,405.346",ellipse:"bessel",datumName:"swiss"},ggrs87:{towgs84:"-199.87,74.79,246.62",ellipse:"GRS80",datumName:"Greek_Geodetic_Reference_System_1987"},nad83:{towgs84:"0,0,0",ellipse:"GRS80",datumName:"North_American_Datum_1983"},nad27:{nadgrids:"@conus,@alaska,@ntv2_0.gsb,@ntv1_can.dat",ellipse:"clrk66",datumName:"North_American_Datum_1927"},potsdam:{towgs84:"606.0,23.0,413.0",ellipse:"bessel",datumName:"Potsdam Rauenberg 1950 DHDN"},carthage:{towgs84:"-263.0,6.0,431.0",ellipse:"clark80",datumName:"Carthage 1934 Tunisia"},hermannskogel:{towgs84:"653.0,-212.0,449.0",ellipse:"bessel",datumName:"Hermannskogel"},osni52:{towgs84:"482.530,-130.596,564.557,-1.042,-0.214,-0.631,8.15",ellipse:"airy",datumName:"Irish National"},ire65:{towgs84:"482.530,-130.596,564.557,-1.042,-0.214,-0.631,8.15",ellipse:"mod_airy",datumName:"Ireland 1965"},rassadiran:{towgs84:"-133.63,-157.5,-158.62",ellipse:"intl",datumName:"Rassadiran"},nzgd49:{towgs84:"59.47,-5.04,187.44,0.47,-0.1,1.024,-4.5993",ellipse:"intl",datumName:"New Zealand Geodetic Datum 1949"},osgb36:{towgs84:"446.448,-125.157,542.060,0.1502,0.2470,0.8421,-20.4894",ellipse:"airy",datumName:"Airy 1830"},s_jtsk:{towgs84:"589,76,480",ellipse:"bessel",datumName:"S-JTSK (Ferro)"},beduaram:{towgs84:"-106,-87,188",ellipse:"clrk80",datumName:"Beduaram"},gunung_segara:{towgs84:"-403,684,41",ellipse:"bessel",datumName:"Gunung Segara Jakarta"},rnb72:{towgs84:"106.869,-52.2978,103.724,-0.33657,0.456955,-1.84218,1",ellipse:"intl",datumName:"Reseau National Belge 1972"}};q.projections=pt,q.projections.start();var xt=function(t,s,i){return h=s,((a=t).datum_type!==h.datum_type||a.a!==h.a||5e-11<Math.abs(a.es-h.es)||(a.datum_type===I?a.datum_params[0]!==h.datum_params[0]||a.datum_params[1]!==h.datum_params[1]||a.datum_params[2]!==h.datum_params[2]:a.datum_type===O&&(a.datum_params[0]!==h.datum_params[0]||a.datum_params[1]!==h.datum_params[1]||a.datum_params[2]!==h.datum_params[2]||a.datum_params[3]!==h.datum_params[3]||a.datum_params[4]!==h.datum_params[4]||a.datum_params[5]!==h.datum_params[5]||a.datum_params[6]!==h.datum_params[6])))&&t.datum_type!==G&&s.datum_type!==G&&(t.es!==s.es||t.a!==s.a||u(t.datum_type)||u(s.datum_type))?(i=M(i,t.es,t.a),u(t.datum_type)&&(i=function(t,s,i){if(s===I)return{x:t.x+i[0],y:t.y+i[1],z:t.z+i[2]};if(s===O){var a=i[0],h=i[1],e=i[2],n=i[3],r=i[4],o=i[5],l=i[6];return{x:l*(t.x-o*t.y+r*t.z)+a,y:l*(o*t.x+t.y-n*t.z)+h,z:l*(-r*t.x+n*t.y+t.z)+e}}}(i,t.datum_type,t.datum_params)),u(s.datum_type)&&(i=function(t,s,i){if(s===I)return{x:t.x-i[0],y:t.y-i[1],z:t.z-i[2]};if(s===O){var a=i[0],h=i[1],e=i[2],n=i[3],r=i[4],o=i[5],l=i[6],M=(t.x-a)/l,c=(t.y-h)/l,u=(t.z-e)/l;return{x:M+o*c-r*u,y:-o*M+c+n*u,z:r*M-n*c+u}}}(i,s.datum_type,s.datum_params)),c(i,s.es,s.a,s.b)):i;var a,h},gt=function(t,s,i){for(var a,h,e=i.x,n=i.y,r=i.z||0,o={},l=0;l<3;l++)if(!s||2!==l||void 0!==i.z)switch(h=0===l?(a=e,-1!=="ew".indexOf(t.axis[l])?"x":"y"):1===l?(a=n,-1!=="ns".indexOf(t.axis[l])?"y":"x"):(a=r,"z"),t.axis[l]){case"e":case"w":case"n":case"s":o[h]=a;break;case"u":void 0!==i[h]&&(o.z=a);break;case"d":void 0!==i[h]&&(o.z=-a);break;default:return null}return o},bt=function(t){var s={x:t[0],y:t[1]};return 2<t.length&&(s.z=t[2]),3<t.length&&(s.m=t[3]),s},vt=function(t){i(t.x),i(t.y)},wt=q("WGS84"),Ct=6,Pt="AJSAJS",St="AFAFAF",Nt=65,kt=73,Et=79,qt=86,It=90,Ot={forward:d,inverse:function(t){var s=g(v(t.toUpperCase()));return s.lat&&s.lon?[s.lon,s.lat,s.lon,s.lat]:[s.left,s.bottom,s.right,s.top]},toPoint:y};C.fromMGRS=function(t){return new C(y(t))},C.prototype.toMGRS=function(t){return d([this.x,this.y],t)};function At(t){var s=[];s[0]=1-t*(.25+t*(.046875+t*(.01953125+t*ts))),s[1]=t*(.75-t*(.046875+t*(.01953125+t*ts)));var i=t*t;return s[2]=i*(.46875-t*(.013020833333333334+.007120768229166667*t)),i*=t,s[3]=i*(.3645833333333333-.005696614583333333*t),s[4]=i*t*.3076171875,s}function Gt(t,s,i,a){return i*=s,s*=s,a[0]*t-i*(a[1]+s*(a[2]+s*(a[3]+s*a[4])))}function jt(t,s,i){for(var a=1/(1-s),h=t,e=20;e;--e){var n=Math.sin(h),r=1-s*n*n;if(h-=r=(Gt(h,n,Math.cos(h),i)-t)*(r*Math.sqrt(r))*a,Math.abs(r)<D)return h}return h}function zt(t){var s=Math.exp(t);return(s-1/s)/2}function Rt(t,s){t=Math.abs(t),s=Math.abs(s);var i=Math.max(t,s),a=Math.min(t,s)/(i||1);return i*Math.sqrt(1+Math.pow(a,2))}function Lt(t){var s,i,a,h=Math.abs(t);return s=h*(1+h/(Rt(1,h)+1)),h=0==(a=(i=1+s)-1)?s:s*Math.log(i)/a,t<0?-h:h}function Tt(t,s){for(var i,a=2*Math.cos(2*s),h=t.length-1,e=t[h],n=0;0<=--h;)i=a*e-n+t[h],n=e,e=i;return s+i*Math.sin(2*s)}function Dt(t,s,i){for(var a,h,e,n,r=Math.sin(s),o=Math.cos(s),l=zt(i),M=(e=i,((n=Math.exp(e))+1/n)/2),c=2*o*M,u=-2*r*l,f=t.length-1,m=t[f],p=0,d=0,y=0;0<=--f;)a=d,h=p,m=c*(d=m)-a-u*(p=y)+t[f],y=u*d-h+c*p;return[(c=r*M)*m-(u=o*l)*y,c*y+u*m]}function Bt(t,s){return Math.pow((1-t)/(1+t),s)}function Ut(t,s,i,a,h){return t*h-s*Math.sin(2*h)+i*Math.sin(4*h)-a*Math.sin(6*h)}function Ft(t){return 1-.25*t*(1+t/16*(3+1.25*t))}function Qt(t){return.375*t*(1+.25*t*(1+.46875*t))}function Wt(t){return.05859375*t*t*(1+.75*t)}function Xt(t){return t*t*t*(35/3072)}function Ht(t,s,i){var a=s*i;return t/Math.sqrt(1-a*a)}function Jt(t){return Math.abs(t)<z?t:t-et(t)*Math.PI}function Kt(t,s,i,a,h){for(var e,n=t/s,r=0;r<15;r++)if(n+=e=(t-(s*n-i*Math.sin(2*n)+a*Math.sin(4*n)-h*Math.sin(6*n)))/(s-2*i*Math.cos(2*n)+4*a*Math.cos(4*n)-6*h*Math.cos(6*n)),Math.abs(e)<=1e-10)return n;return NaN}function Vt(t,s){var i;return 1e-7<t?(1-t*t)*(s/(1-(i=t*s)*i)-.5/t*Math.log((1-i)/(1+i))):2*s}function Zt(t){return 1<Math.abs(t)&&(t=1<t?1:-1),Math.asin(t)}function Yt(t,s){return t[0]+s*(t[1]+s*(t[2]+s*t[3]))}var $t,ts=.01068115234375,ss={init:function(){this.x0=void 0!==this.x0?this.x0:0,this.y0=void 0!==this.y0?this.y0:0,this.long0=void 0!==this.long0?this.long0:0,this.lat0=void 0!==this.lat0?this.lat0:0,this.es&&(this.en=At(this.es),this.ml0=Gt(this.lat0,Math.sin(this.lat0),Math.cos(this.lat0),this.en))},forward:function(t){var s=t.x,i=t.y,a=nt(s-this.long0),h=Math.sin(i),e=Math.cos(i);if(this.es){var n=e*a,r=Math.pow(n,2),o=this.ep2*Math.pow(e,2),l=Math.pow(o,2),M=Math.abs(e)>D?Math.tan(i):0,c=Math.pow(M,2),u=Math.pow(c,2),f=1-this.es*Math.pow(h,2);n/=Math.sqrt(f);var m=Gt(i,h,e,this.en),p=this.a*(this.k0*n*(1+r/6*(1-c+o+r/20*(5-18*c+u+14*o-58*c*o+r/42*(61+179*u-u*c-479*c)))))+this.x0,d=this.a*(this.k0*(m-this.ml0+h*a*n/2*(1+r/12*(5-c+9*o+4*l+r/30*(61+u-58*c+270*o-330*c*o+r/56*(1385+543*u-u*c-3111*c))))))+this.y0}else{var y=e*Math.sin(a);if(Math.abs(Math.abs(y)-1)<D)return 93;if(p=.5*this.a*this.k0*Math.log((1+y)/(1-y))+this.x0,d=e*Math.cos(a)/Math.sqrt(1-Math.pow(y,2)),1<=(y=Math.abs(d))){if(D<y-1)return 93;d=0}else d=Math.acos(d);i<0&&(d=-d),d=this.a*this.k0*(d-this.lat0)+this.y0}return t.x=p,t.y=d,t},inverse:function(t){var s,i,a,h,e,n,r,o,l,M,c,u,f,m,p,d,y,_=(t.x-this.x0)*(1/this.a),x=(t.y-this.y0)*(1/this.a);return f=this.es?(l=this.ml0+x/this.k0,s=jt(l,this.es,this.en),Math.abs(s)<z?(i=Math.sin(s),a=Math.cos(s),h=Math.abs(a)>D?Math.tan(s):0,e=this.ep2*Math.pow(a,2),n=Math.pow(e,2),r=Math.pow(h,2),o=Math.pow(r,2),l=1-this.es*Math.pow(i,2),M=_*Math.sqrt(l)/this.k0,u=s-(l*=h)*(c=Math.pow(M,2))/(1-this.es)*.5*(1-c/12*(5+3*r-9*e*r+e-4*n-c/30*(61+90*r-252*e*r+45*o+46*e-c/56*(1385+3633*r+4095*o+1574*o*r)))),nt(this.long0+M*(1-c/6*(1+2*r+e-c/20*(5+28*r+24*o+8*e*r+6*e-c/42*(61+662*r+1320*o+720*o*r))))/a)):(u=z*et(x),0)):(p=.5*((m=Math.exp(_/this.k0))-1/m),d=this.lat0+x/this.k0,y=Math.cos(d),l=Math.sqrt((1-Math.pow(y,2))/(1+Math.pow(p,2))),u=Math.asin(l),x<0&&(u=-u),0==p&&0===y?0:nt(Math.atan2(p,y)+this.long0)),t.x=f,t.y=u,t},names:["Transverse_Mercator","Transverse Mercator","tmerc"]},is={init:function(){if(void 0===this.es||this.es<=0)throw new Error("incorrect elliptical usage");this.x0=void 0!==this.x0?this.x0:0,this.y0=void 0!==this.y0?this.y0:0,this.long0=void 0!==this.long0?this.long0:0,this.lat0=void 0!==this.lat0?this.lat0:0,this.cgb=[],this.cbg=[],this.utg=[],this.gtu=[];var t=this.es/(1+Math.sqrt(1-this.es)),s=t/(2-t),i=s;this.cgb[0]=s*(2+s*(-2/3+s*(s*(116/45+s*(26/45+-2854/675*s))-2))),this.cbg[0]=s*(s*(2/3+s*(4/3+s*(-82/45+s*(32/45+4642/4725*s))))-2),i*=s,this.cgb[1]=i*(7/3+s*(s*(-227/45+s*(2704/315+2323/945*s))-1.6)),this.cbg[1]=i*(5/3+s*(-16/15+s*(-13/9+s*(904/315+-1522/945*s)))),i*=s,this.cgb[2]=i*(56/15+s*(-136/35+s*(-1262/105+73814/2835*s))),this.cbg[2]=i*(-26/15+s*(34/21+s*(1.6+-12686/2835*s))),i*=s,this.cgb[3]=i*(4279/630+s*(-332/35+-399572/14175*s)),this.cbg[3]=i*(1237/630+s*(-24832/14175*s-2.4)),i*=s,this.cgb[4]=i*(4174/315+-144838/6237*s),this.cbg[4]=i*(-734/315+109598/31185*s),i*=s,this.cgb[5]=i*(601676/22275),this.cbg[5]=i*(444337/155925),i=Math.pow(s,2),this.Qn=this.k0/(1+s)*(1+i*(.25+i*(1/64+i/256))),this.utg[0]=s*(s*(2/3+s*(-37/96+s*(1/360+s*(81/512+-96199/604800*s))))-.5),this.gtu[0]=s*(.5+s*(-2/3+s*(5/16+s*(41/180+s*(-127/288+7891/37800*s))))),this.utg[1]=i*(-1/48+s*(-1/15+s*(437/1440+s*(-46/105+1118711/3870720*s)))),this.gtu[1]=i*(13/48+s*(s*(557/1440+s*(281/630+-1983433/1935360*s))-.6)),i*=s,this.utg[2]=i*(-17/480+s*(37/840+s*(209/4480+-5569/90720*s))),this.gtu[2]=i*(61/240+s*(-103/140+s*(15061/26880+167603/181440*s))),i*=s,this.utg[3]=i*(-4397/161280+s*(11/504+830251/7257600*s)),this.gtu[3]=i*(49561/161280+s*(-179/168+6601661/7257600*s)),i*=s,this.utg[4]=i*(-4583/161280+108847/3991680*s),this.gtu[4]=i*(34729/80640+-3418889/1995840*s),i*=s,this.utg[5]=-.03233083094085698*i,this.gtu[5]=.6650675310896665*i;var a=Tt(this.cbg,this.lat0);this.Zb=-this.Qn*(a+function(t,s){for(var i,a=2*Math.cos(s),h=t.length-1,e=t[h],n=0;0<=--h;)i=a*e-n+t[h],n=e,e=i;return Math.sin(s)*i}(this.gtu,2*a))},forward:function(t){var s=nt(t.x-this.long0),i=t.y,i=Tt(this.cbg,i),a=Math.sin(i),h=Math.cos(i),e=Math.sin(s),n=Math.cos(s);i=Math.atan2(a,n*h),s=Math.atan2(e*h,Rt(a,h*n)),s=Lt(Math.tan(s));var r,o,l=Dt(this.gtu,2*i,2*s);return i+=l[0],s+=l[1],o=Math.abs(s)<=2.623395162778?(r=this.a*(this.Qn*s)+this.x0,this.a*(this.Qn*i+this.Zb)+this.y0):r=1/0,t.x=r,t.y=o,t},inverse:function(t){var s,i,a,h,e,n,r,o=(t.x-this.x0)*(1/this.a),l=(t.y-this.y0)*(1/this.a);return l=(l-this.Zb)/this.Qn,o/=this.Qn,r=Math.abs(o)<=2.623395162778?(l+=(s=Dt(this.utg,2*l,2*o))[0],o+=s[1],o=Math.atan(zt(o)),i=Math.sin(l),a=Math.cos(l),h=Math.sin(o),e=Math.cos(o),l=Math.atan2(i*e,Rt(h,e*a)),o=Math.atan2(h,e*a),n=nt(o+this.long0),Tt(this.cgb,l)):n=1/0,t.x=n,t.y=r,t},names:["Extended_Transverse_Mercator","Extended Transverse Mercator","etmerc"]},as={init:function(){var t=function(t,s){if(void 0===t){if((t=Math.floor(30*(nt(s)+Math.PI)/Math.PI)+1)<0)return 0;if(60<t)return 60}return t}(this.zone,this.long0);if(void 0===t)throw new Error("unknown utm zone");this.lat0=0,this.long0=(6*Math.abs(t)-183)*N,this.x0=5e5,this.y0=this.utmSouth?1e7:0,this.k0=.9996,is.init.apply(this),this.forward=is.forward,this.inverse=is.inverse},names:["Universal Transverse Mercator System","utm"],dependsOn:"etmerc"},hs={init:function(){var t=Math.sin(this.lat0),s=Math.cos(this.lat0);s*=s,this.rc=Math.sqrt(1-this.es)/(1-this.es*t*t),this.C=Math.sqrt(1+this.es*s*s/(1-this.es)),this.phic0=Math.asin(t/this.C),this.ratexp=.5*this.C*this.e,this.K=Math.tan(.5*this.phic0+U)/(Math.pow(Math.tan(.5*this.lat0+U),this.C)*Bt(this.e*t,this.ratexp))},forward:function(t){var s=t.x,i=t.y;return t.y=2*Math.atan(this.K*Math.pow(Math.tan(.5*i+U),this.C)*Bt(this.e*Math.sin(i),this.ratexp))-z,t.x=this.C*s,t},inverse:function(t){for(var s=t.x/this.C,i=t.y,a=Math.pow(Math.tan(.5*i+U)/this.K,1/this.C),h=20;0<h&&(i=2*Math.atan(a*Bt(this.e*Math.sin(t.y),-.5*this.e))-z,!(Math.abs(i-t.y)<1e-14));--h)t.y=i;return h?(t.x=s,t.y=i,t):null},names:["gauss"]},es={init:function(){hs.init.apply(this),this.rc&&(this.sinc0=Math.sin(this.phic0),this.cosc0=Math.cos(this.phic0),this.R2=2*this.rc,this.title||(this.title="Oblique Stereographic Alternative"))},forward:function(t){var s,i,a,h;return t.x=nt(t.x-this.long0),hs.forward.apply(this,[t]),s=Math.sin(t.y),i=Math.cos(t.y),a=Math.cos(t.x),h=this.k0*this.R2/(1+this.sinc0*s+this.cosc0*i*a),t.x=h*i*Math.sin(t.x),t.y=h*(this.cosc0*s-this.sinc0*i*a),t.x=this.a*t.x+this.x0,t.y=this.a*t.y+this.y0,t},inverse:function(t){var s,i,a,h,e,n;return t.x=(t.x-this.x0)/this.a,t.y=(t.y-this.y0)/this.a,t.x/=this.k0,t.y/=this.k0,n=(s=Math.sqrt(t.x*t.x+t.y*t.y))?(i=2*Math.atan2(s,this.R2),a=Math.sin(i),h=Math.cos(i),e=Math.asin(h*this.sinc0+t.y*a*this.cosc0/s),Math.atan2(t.x*a,s*this.cosc0*h-t.y*this.sinc0*a)):(e=this.phic0,0),t.x=n,t.y=e,hs.inverse.apply(this,[t]),t.x=nt(t.x+this.long0),t},names:["Stereographic_North_Pole","Oblique_Stereographic","Polar_Stereographic","sterea","Oblique Stereographic Alternative","Double_Stereographic"]},ns={init:function(){this.coslat0=Math.cos(this.lat0),this.sinlat0=Math.sin(this.lat0),this.sphere?1===this.k0&&!isNaN(this.lat_ts)&&Math.abs(this.coslat0)<=D&&(this.k0=.5*(1+et(this.lat0)*Math.sin(this.lat_ts))):(Math.abs(this.coslat0)<=D&&(0<this.lat0?this.con=1:this.con=-1),this.cons=Math.sqrt(Math.pow(1+this.e,1+this.e)*Math.pow(1-this.e,1-this.e)),1===this.k0&&!isNaN(this.lat_ts)&&Math.abs(this.coslat0)<=D&&(this.k0=.5*this.cons*ht(this.e,Math.sin(this.lat_ts),Math.cos(this.lat_ts))/rt(this.e,this.con*this.lat_ts,this.con*Math.sin(this.lat_ts))),this.ms1=ht(this.e,this.sinlat0,this.coslat0),this.X0=2*Math.atan(this.ssfn_(this.lat0,this.sinlat0,this.e))-z,this.cosX0=Math.cos(this.X0),this.sinX0=Math.sin(this.X0))},forward:function(t){var s,i,a,h,e,n,r=t.x,o=t.y,l=Math.sin(o),M=Math.cos(o),c=nt(r-this.long0);return Math.abs(Math.abs(r-this.long0)-Math.PI)<=D&&Math.abs(o+this.lat0)<=D?(t.x=NaN,t.y=NaN):this.sphere?(s=2*this.k0/(1+this.sinlat0*l+this.coslat0*M*Math.cos(c)),t.x=this.a*s*M*Math.sin(c)+this.x0,t.y=this.a*s*(this.coslat0*l-this.sinlat0*M*Math.cos(c))+this.y0):(i=2*Math.atan(this.ssfn_(o,l,this.e))-z,h=Math.cos(i),a=Math.sin(i),Math.abs(this.coslat0)<=D?(e=rt(this.e,o*this.con,this.con*l),n=2*this.a*this.k0*e/this.cons,t.x=this.x0+n*Math.sin(r-this.long0),t.y=this.y0-this.con*n*Math.cos(r-this.long0)):(Math.abs(this.sinlat0)<D?(s=2*this.a*this.k0/(1+h*Math.cos(c)),t.y=s*a):(s=2*this.a*this.k0*this.ms1/(this.cosX0*(1+this.sinX0*a+this.cosX0*h*Math.cos(c))),t.y=s*(this.cosX0*a-this.sinX0*h*Math.cos(c))+this.y0),t.x=s*h*Math.sin(c)+this.x0)),t},inverse:function(t){t.x-=this.x0,t.y-=this.y0;var s,i,a,h=Math.sqrt(t.x*t.x+t.y*t.y);if(this.sphere){var e=2*Math.atan(h/(2*this.a*this.k0)),n=this.long0,r=this.lat0;return h<=D||(r=Math.asin(Math.cos(e)*this.sinlat0+t.y*Math.sin(e)*this.coslat0/h),n=nt(Math.abs(this.coslat0)<D?0<this.lat0?this.long0+Math.atan2(t.x,-1*t.y):this.long0+Math.atan2(t.x,t.y):this.long0+Math.atan2(t.x*Math.sin(e),h*this.coslat0*Math.cos(e)-t.y*this.sinlat0*Math.sin(e)))),t.x=n,t.y=r,t}if(Math.abs(this.coslat0)<=D){if(h<=D)return r=this.lat0,n=this.long0,t.x=n,t.y=r,t;t.x*=this.con,t.y*=this.con,s=h*this.cons/(2*this.a*this.k0),r=this.con*ot(this.e,s),n=this.con*nt(this.con*this.long0+Math.atan2(t.x,-1*t.y))}else i=2*Math.atan(h*this.cosX0/(2*this.a*this.k0*this.ms1)),n=this.long0,h<=D?a=this.X0:(a=Math.asin(Math.cos(i)*this.sinX0+t.y*Math.sin(i)*this.cosX0/h),n=nt(this.long0+Math.atan2(t.x*Math.sin(i),h*this.cosX0*Math.cos(i)-t.y*this.sinX0*Math.sin(i)))),r=-1*ot(this.e,Math.tan(.5*(z+a)));return t.x=n,t.y=r,t},names:["stere","Stereographic_South_Pole","Polar Stereographic (variant B)"],ssfn_:function(t,s,i){return s*=i,Math.tan(.5*(z+t))*Math.pow((1-s)/(1+s),.5*i)}},rs={init:function(){var t=this.lat0;this.lambda0=this.long0;var s=Math.sin(t),i=this.a,a=1/this.rf,h=2*a-Math.pow(a,2),e=this.e=Math.sqrt(h);this.R=this.k0*i*Math.sqrt(1-h)/(1-h*Math.pow(s,2)),this.alpha=Math.sqrt(1+h/(1-h)*Math.pow(Math.cos(t),4)),this.b0=Math.asin(s/this.alpha);var n=Math.log(Math.tan(Math.PI/4+this.b0/2)),r=Math.log(Math.tan(Math.PI/4+t/2)),o=Math.log((1+e*s)/(1-e*s));this.K=n-this.alpha*r+this.alpha*e/2*o},forward:function(t){var s=Math.log(Math.tan(Math.PI/4-t.y/2)),i=this.e/2*Math.log((1+this.e*Math.sin(t.y))/(1-this.e*Math.sin(t.y))),a=-this.alpha*(s+i)+this.K,h=2*(Math.atan(Math.exp(a))-Math.PI/4),e=this.alpha*(t.x-this.lambda0),n=Math.atan(Math.sin(e)/(Math.sin(this.b0)*Math.tan(h)+Math.cos(this.b0)*Math.cos(e))),r=Math.asin(Math.cos(this.b0)*Math.sin(h)-Math.sin(this.b0)*Math.cos(h)*Math.cos(e));return t.y=this.R/2*Math.log((1+Math.sin(r))/(1-Math.sin(r)))+this.y0,t.x=this.R*n+this.x0,t},inverse:function(t){for(var s=t.x-this.x0,i=t.y-this.y0,a=s/this.R,h=2*(Math.atan(Math.exp(i/this.R))-Math.PI/4),e=Math.asin(Math.cos(this.b0)*Math.sin(h)+Math.sin(this.b0)*Math.cos(h)*Math.cos(a)),n=Math.atan(Math.sin(a)/(Math.cos(this.b0)*Math.cos(a)-Math.sin(this.b0)*Math.tan(h))),r=this.lambda0+n/this.alpha,o=0,l=e,M=-1e3,c=0;1e-7<Math.abs(l-M);){if(20<++c)return;o=1/this.alpha*(Math.log(Math.tan(Math.PI/4+e/2))-this.K)+this.e*Math.log(Math.tan(Math.PI/4+Math.asin(this.e*Math.sin(l))/2)),M=l,l=2*Math.atan(Math.exp(o))-Math.PI/2}return t.x=r,t.y=l,t},names:["somerc"]},os={init:function(){this.no_off=this.no_off||!1,this.no_rot=this.no_rot||!1,isNaN(this.k0)&&(this.k0=1);var t=Math.sin(this.lat0),s=Math.cos(this.lat0),i=this.e*t;this.bl=Math.sqrt(1+this.es/(1-this.es)*Math.pow(s,4)),this.al=this.a*this.bl*this.k0*Math.sqrt(1-this.es)/(1-i*i);var a,h,e,n,r,o,l,M,c,u,f=rt(this.e,this.lat0,t),m=this.bl/s*Math.sqrt((1-this.es)/(1-i*i));m*m<1&&(m=1),isNaN(this.longc)?(h=rt(this.e,this.lat1,Math.sin(this.lat1)),e=rt(this.e,this.lat2,Math.sin(this.lat2)),0<=this.lat0?this.el=(m+Math.sqrt(m*m-1))*Math.pow(f,this.bl):this.el=(m-Math.sqrt(m*m-1))*Math.pow(f,this.bl),n=Math.pow(h,this.bl),r=Math.pow(e,this.bl),o=.5*((a=this.el/n)-1/a),l=(this.el*this.el-r*n)/(this.el*this.el+r*n),M=(r-n)/(r+n),c=nt(this.long1-this.long2),this.long0=.5*(this.long1+this.long2)-Math.atan(l*Math.tan(.5*this.bl*c)/M)/this.bl,this.long0=nt(this.long0),u=nt(this.long1-this.long0),this.gamma0=Math.atan(Math.sin(this.bl*u)/o),this.alpha=Math.asin(m*Math.sin(this.gamma0))):(a=0<=this.lat0?m+Math.sqrt(m*m-1):m-Math.sqrt(m*m-1),this.el=a*Math.pow(f,this.bl),o=.5*(a-1/a),this.gamma0=Math.asin(Math.sin(this.alpha)/m),this.long0=this.longc-Math.asin(o*Math.tan(this.gamma0))/this.bl),this.no_off?this.uc=0:0<=this.lat0?this.uc=this.al/this.bl*Math.atan2(Math.sqrt(m*m-1),Math.cos(this.alpha)):this.uc=-1*this.al/this.bl*Math.atan2(Math.sqrt(m*m-1),Math.cos(this.alpha))},forward:function(t){var s,i,a,h,e,n,r,o,l,M=t.x,c=t.y,u=nt(M-this.long0);return l=Math.abs(Math.abs(c)-z)<=D?(s=0<c?-1:1,o=this.al/this.bl*Math.log(Math.tan(U+s*this.gamma0*.5)),-1*s*z*this.al/this.bl):(i=rt(this.e,c,Math.sin(c)),h=.5*((a=this.el/Math.pow(i,this.bl))-1/a),e=.5*(a+1/a),n=Math.sin(this.bl*u),r=(h*Math.sin(this.gamma0)-n*Math.cos(this.gamma0))/e,o=Math.abs(Math.abs(r)-1)<=D?Number.POSITIVE_INFINITY:.5*this.al*Math.log((1-r)/(1+r))/this.bl,Math.abs(Math.cos(this.bl*u))<=D?this.al*this.bl*u:this.al*Math.atan2(h*Math.cos(this.gamma0)+n*Math.sin(this.gamma0),Math.cos(this.bl*u))/this.bl),this.no_rot?(t.x=this.x0+l,t.y=this.y0+o):(l-=this.uc,t.x=this.x0+o*Math.cos(this.alpha)+l*Math.sin(this.alpha),t.y=this.y0+l*Math.cos(this.alpha)-o*Math.sin(this.alpha)),t},inverse:function(t){var s,i;this.no_rot?(i=t.y-this.y0,s=t.x-this.x0):(i=(t.x-this.x0)*Math.cos(this.alpha)-(t.y-this.y0)*Math.sin(this.alpha),s=(t.y-this.y0)*Math.cos(this.alpha)+(t.x-this.x0)*Math.sin(this.alpha),s+=this.uc);var a=Math.exp(-1*this.bl*i/this.al),h=.5*(a-1/a),e=.5*(a+1/a),n=Math.sin(this.bl*s/this.al),r=(n*Math.cos(this.gamma0)+h*Math.sin(this.gamma0))/e,o=Math.pow(this.el/Math.sqrt((1+r)/(1-r)),1/this.bl);return Math.abs(r-1)<D?(t.x=this.long0,t.y=z):Math.abs(1+r)<D?(t.x=this.long0,t.y=-1*z):(t.y=ot(this.e,o),t.x=nt(this.long0-Math.atan2(h*Math.cos(this.gamma0)-n*Math.sin(this.gamma0),Math.cos(this.bl*s/this.al))/this.bl)),t},names:["Hotine_Oblique_Mercator","Hotine Oblique Mercator","Hotine_Oblique_Mercator_Azimuth_Natural_Origin","Hotine_Oblique_Mercator_Azimuth_Center","omerc"]},ls={init:function(){var t,s,i,a,h,e,n,r,o,l;this.lat2||(this.lat2=this.lat1),this.k0||(this.k0=1),this.x0=this.x0||0,this.y0=this.y0||0,Math.abs(this.lat1+this.lat2)<D||(t=this.b/this.a,this.e=Math.sqrt(1-t*t),s=Math.sin(this.lat1),i=Math.cos(this.lat1),a=ht(this.e,s,i),h=rt(this.e,this.lat1,s),e=Math.sin(this.lat2),n=Math.cos(this.lat2),r=ht(this.e,e,n),o=rt(this.e,this.lat2,e),l=rt(this.e,this.lat0,Math.sin(this.lat0)),Math.abs(this.lat1-this.lat2)>D?this.ns=Math.log(a/r)/Math.log(h/o):this.ns=s,isNaN(this.ns)&&(this.ns=s),this.f0=a/(this.ns*Math.pow(h,this.ns)),this.rh=this.a*this.f0*Math.pow(l,this.ns),this.title||(this.title="Lambert Conformal Conic"))},forward:function(t){var s=t.x,i=t.y;Math.abs(2*Math.abs(i)-Math.PI)<=D&&(i=et(i)*(z-2*D));var a,h,e=Math.abs(Math.abs(i)-z);if(D<e)a=rt(this.e,i,Math.sin(i)),h=this.a*this.f0*Math.pow(a,this.ns);else{if((e=i*this.ns)<=0)return null;h=0}var n=this.ns*nt(s-this.long0);return t.x=this.k0*(h*Math.sin(n))+this.x0,t.y=this.k0*(this.rh-h*Math.cos(n))+this.y0,t},inverse:function(t){var s,i,a,h,e=(t.x-this.x0)/this.k0,n=this.rh-(t.y-this.y0)/this.k0,r=0<this.ns?(s=Math.sqrt(e*e+n*n),1):(s=-Math.sqrt(e*e+n*n),-1),o=0;if(0!==s&&(o=Math.atan2(r*e,r*n)),0!==s||0<this.ns){if(r=1/this.ns,i=Math.pow(s/(this.a*this.f0),r),-9999===(a=ot(this.e,i)))return null}else a=-z;return h=nt(o/this.ns+this.long0),t.x=h,t.y=a,t},names:["Lambert Tangential Conformal Conic Projection","Lambert_Conformal_Conic","Lambert_Conformal_Conic_2SP","lcc"]},Ms={init:function(){this.a=6377397.155,this.es=.006674372230614,this.e=Math.sqrt(this.es),this.lat0||(this.lat0=.863937979737193),this.long0||(this.long0=.4334234309119251),this.k0||(this.k0=.9999),this.s45=.785398163397448,this.s90=2*this.s45,this.fi0=this.lat0,this.e2=this.es,this.e=Math.sqrt(this.e2),this.alfa=Math.sqrt(1+this.e2*Math.pow(Math.cos(this.fi0),4)/(1-this.e2)),this.uq=1.04216856380474,this.u0=Math.asin(Math.sin(this.fi0)/this.alfa),this.g=Math.pow((1+this.e*Math.sin(this.fi0))/(1-this.e*Math.sin(this.fi0)),this.alfa*this.e/2),this.k=Math.tan(this.u0/2+this.s45)/Math.pow(Math.tan(this.fi0/2+this.s45),this.alfa)*this.g,this.k1=this.k0,this.n0=this.a*Math.sqrt(1-this.e2)/(1-this.e2*Math.pow(Math.sin(this.fi0),2)),this.s0=1.37008346281555,this.n=Math.sin(this.s0),this.ro0=this.k1*this.n0/Math.tan(this.s0),this.ad=this.s90-this.uq},forward:function(t){var s=t.x,i=t.y,a=nt(s-this.long0),h=Math.pow((1+this.e*Math.sin(i))/(1-this.e*Math.sin(i)),this.alfa*this.e/2),e=2*(Math.atan(this.k*Math.pow(Math.tan(i/2+this.s45),this.alfa)/h)-this.s45),n=-a*this.alfa,r=Math.asin(Math.cos(this.ad)*Math.sin(e)+Math.sin(this.ad)*Math.cos(e)*Math.cos(n)),o=Math.asin(Math.cos(e)*Math.sin(n)/Math.cos(r)),l=this.n*o,M=this.ro0*Math.pow(Math.tan(this.s0/2+this.s45),this.n)/Math.pow(Math.tan(r/2+this.s45),this.n);return t.y=M*Math.cos(l),t.x=M*Math.sin(l),this.czech||(t.y*=-1,t.x*=-1),t},inverse:function(t){var s,i,a,h,e,n,r,o=t.x;t.x=t.y,t.y=o,this.czech||(t.y*=-1,t.x*=-1),e=Math.sqrt(t.x*t.x+t.y*t.y),h=Math.atan2(t.y,t.x)/Math.sin(this.s0),a=2*(Math.atan(Math.pow(this.ro0/e,1/this.n)*Math.tan(this.s0/2+this.s45))-this.s45),s=Math.asin(Math.cos(this.ad)*Math.sin(a)-Math.sin(this.ad)*Math.cos(a)*Math.cos(h)),i=Math.asin(Math.cos(a)*Math.sin(h)/Math.cos(s)),t.x=this.long0-i/this.alfa,n=s;for(var l=r=0;t.y=2*(Math.atan(Math.pow(this.k,-1/this.alfa)*Math.pow(Math.tan(s/2+this.s45),1/this.alfa)*Math.pow((1+this.e*Math.sin(n))/(1-this.e*Math.sin(n)),this.e/2))-this.s45),Math.abs(n-t.y)<1e-10&&(r=1),n=t.y,l+=1,0===r&&l<15;);return 15<=l?null:t},names:["Krovak","krovak"]},cs={init:function(){this.sphere||(this.e0=Ft(this.es),this.e1=Qt(this.es),this.e2=Wt(this.es),this.e3=Xt(this.es),this.ml0=this.a*Ut(this.e0,this.e1,this.e2,this.e3,this.lat0))},forward:function(t){var s,i,a,h,e,n,r,o,l,M=t.x,c=t.y,M=nt(M-this.long0);return l=this.sphere?(o=this.a*Math.asin(Math.cos(c)*Math.sin(M)),this.a*(Math.atan2(Math.tan(c),Math.cos(M))-this.lat0)):(s=Math.sin(c),i=Math.cos(c),a=Ht(this.a,this.e,s),h=Math.tan(c)*Math.tan(c),o=a*(e=M*Math.cos(c))*(1-(n=e*e)*h*(1/6-(8-h+8*(r=this.es*i*i/(1-this.es)))*n/120)),this.a*Ut(this.e0,this.e1,this.e2,this.e3,c)-this.ml0+a*s/i*n*(.5+(5-h+6*r)*n/24)),t.x=o+this.x0,t.y=l+this.y0,t},inverse:function(t){t.x-=this.x0,t.y-=this.y0;var s=t.x/this.a,i=t.y/this.a;if(this.sphere)var a=i+this.lat0,h=Math.asin(Math.sin(a)*Math.cos(s)),e=Math.atan2(Math.tan(s),Math.cos(a));else{var n=this.ml0/this.a+i,r=Kt(n,this.e0,this.e1,this.e2,this.e3);if(Math.abs(Math.abs(r)-z)<=D)return t.x=this.long0,t.y=z,i<0&&(t.y*=-1),t;var o=Ht(this.a,this.e,Math.sin(r)),l=o*o*o/this.a/this.a*(1-this.es),M=Math.pow(Math.tan(r),2),c=s*this.a/o,u=c*c;h=r-o*Math.tan(r)/l*c*c*(.5-(1+3*M)*c*c/24),e=c*(1-u*(M/3+(1+3*M)*M*u/15))/Math.cos(r)}return t.x=nt(e+this.long0),t.y=Jt(h),t},names:["Cassini","Cassini_Soldner","cass"]},us={init:function(){var t,s,i,a,h=Math.abs(this.lat0);if(Math.abs(h-z)<D?this.mode=this.lat0<0?this.S_POLE:this.N_POLE:Math.abs(h)<D?this.mode=this.EQUIT:this.mode=this.OBLIQ,0<this.es)switch(this.qp=Vt(this.e,1),this.mmf=.5/(1-this.es),this.apa=(s=this.es,(a=[])[0]=.3333333333333333*s,i=s*s,a[0]+=.17222222222222222*i,a[1]=.06388888888888888*i,i*=s,a[0]+=.10257936507936508*i,a[1]+=.0664021164021164*i,a[2]=.016415012942191543*i,a),this.mode){case this.N_POLE:case this.S_POLE:this.dd=1;break;case this.EQUIT:this.rq=Math.sqrt(.5*this.qp),this.dd=1/this.rq,this.xmf=1,this.ymf=.5*this.qp;break;case this.OBLIQ:this.rq=Math.sqrt(.5*this.qp),t=Math.sin(this.lat0),this.sinb1=Vt(this.e,t)/this.qp,this.cosb1=Math.sqrt(1-this.sinb1*this.sinb1),this.dd=Math.cos(this.lat0)/(Math.sqrt(1-this.es*t*t)*this.rq*this.cosb1),this.ymf=(this.xmf=this.rq)/this.dd,this.xmf*=this.dd}else this.mode===this.OBLIQ&&(this.sinph0=Math.sin(this.lat0),this.cosph0=Math.cos(this.lat0))},forward:function(t){var s,i,a,h,e,n,r,o,l,M,c=t.x,u=t.y,c=nt(c-this.long0);if(this.sphere){if(e=Math.sin(u),M=Math.cos(u),a=Math.cos(c),this.mode===this.OBLIQ||this.mode===this.EQUIT){if((i=this.mode===this.EQUIT?1+M*a:1+this.sinph0*e+this.cosph0*M*a)<=D)return null;s=(i=Math.sqrt(2/i))*M*Math.sin(c),i*=this.mode===this.EQUIT?e:this.cosph0*e-this.sinph0*M*a}else if(this.mode===this.N_POLE||this.mode===this.S_POLE){if(this.mode===this.N_POLE&&(a=-a),Math.abs(u+this.lat0)<D)return null;i=U-.5*u,s=(i=2*(this.mode===this.S_POLE?Math.cos(i):Math.sin(i)))*Math.sin(c),i*=a}}else{switch(l=o=r=0,a=Math.cos(c),h=Math.sin(c),e=Math.sin(u),n=Vt(this.e,e),this.mode!==this.OBLIQ&&this.mode!==this.EQUIT||(r=n/this.qp,o=Math.sqrt(1-r*r)),this.mode){case this.OBLIQ:l=1+this.sinb1*r+this.cosb1*o*a;break;case this.EQUIT:l=1+o*a;break;case this.N_POLE:l=z+u,n=this.qp-n;break;case this.S_POLE:l=u-z,n=this.qp+n}if(Math.abs(l)<D)return null;switch(this.mode){case this.OBLIQ:case this.EQUIT:l=Math.sqrt(2/l),i=this.mode===this.OBLIQ?this.ymf*l*(this.cosb1*r-this.sinb1*o*a):(l=Math.sqrt(2/(1+o*a)))*r*this.ymf,s=this.xmf*l*o*h;break;case this.N_POLE:case this.S_POLE:0<=n?(s=(l=Math.sqrt(n))*h,i=a*(this.mode===this.S_POLE?l:-l)):s=i=0}}return t.x=this.a*s+this.x0,t.y=this.a*i+this.y0,t},inverse:function(t){t.x-=this.x0,t.y-=this.y0;var s,i,a,h,e,n,r,o,l,M,c=t.x/this.a,u=t.y/this.a;if(this.sphere){var f=0,m=0,p=Math.sqrt(c*c+u*u);if(1<(i=.5*p))return null;switch(i=2*Math.asin(i),this.mode!==this.OBLIQ&&this.mode!==this.EQUIT||(m=Math.sin(i),f=Math.cos(i)),this.mode){case this.EQUIT:i=Math.abs(p)<=D?0:Math.asin(u*m/p),c*=m,u=f*p;break;case this.OBLIQ:i=Math.abs(p)<=D?this.lat0:Math.asin(f*this.sinph0+u*m*this.cosph0/p),c*=m*this.cosph0,u=(f-Math.sin(i)*this.sinph0)*p;break;case this.N_POLE:u=-u,i=z-i;break;case this.S_POLE:i-=z}s=0!==u||this.mode!==this.EQUIT&&this.mode!==this.OBLIQ?Math.atan2(c,u):0}else{if(r=0,this.mode===this.OBLIQ||this.mode===this.EQUIT){if(c/=this.dd,u*=this.dd,(n=Math.sqrt(c*c+u*u))<D)return t.x=this.long0,t.y=this.lat0,t;h=2*Math.asin(.5*n/this.rq),a=Math.cos(h),c*=h=Math.sin(h),u=this.mode===this.OBLIQ?(r=a*this.sinb1+u*h*this.cosb1/n,e=this.qp*r,n*this.cosb1*a-u*this.sinb1*h):(r=u*h/n,e=this.qp*r,n*a)}else if(this.mode===this.N_POLE||this.mode===this.S_POLE){if(this.mode===this.N_POLE&&(u=-u),!(e=c*c+u*u))return t.x=this.long0,t.y=this.lat0,t;r=1-e/this.qp,this.mode===this.S_POLE&&(r=-r)}s=Math.atan2(c,u),o=Math.asin(r),l=this.apa,M=o+o,i=o+l[0]*Math.sin(M)+l[1]*Math.sin(M+M)+l[2]*Math.sin(M+M+M)}return t.x=nt(this.long0+s),t.y=i,t},names:["Lambert Azimuthal Equal Area","Lambert_Azimuthal_Equal_Area","laea"],S_POLE:1,N_POLE:2,EQUIT:3,OBLIQ:4},fs={init:function(){Math.abs(this.lat1+this.lat2)<D||(this.temp=this.b/this.a,this.es=1-Math.pow(this.temp,2),this.e3=Math.sqrt(this.es),this.sin_po=Math.sin(this.lat1),this.cos_po=Math.cos(this.lat1),this.t1=this.sin_po,this.con=this.sin_po,this.ms1=ht(this.e3,this.sin_po,this.cos_po),this.qs1=Vt(this.e3,this.sin_po,this.cos_po),this.sin_po=Math.sin(this.lat2),this.cos_po=Math.cos(this.lat2),this.t2=this.sin_po,this.ms2=ht(this.e3,this.sin_po,this.cos_po),this.qs2=Vt(this.e3,this.sin_po,this.cos_po),this.sin_po=Math.sin(this.lat0),this.cos_po=Math.cos(this.lat0),this.t3=this.sin_po,this.qs0=Vt(this.e3,this.sin_po,this.cos_po),Math.abs(this.lat1-this.lat2)>D?this.ns0=(this.ms1*this.ms1-this.ms2*this.ms2)/(this.qs2-this.qs1):this.ns0=this.con,this.c=this.ms1*this.ms1+this.ns0*this.qs1,this.rh=this.a*Math.sqrt(this.c-this.ns0*this.qs0)/this.ns0)},forward:function(t){var s=t.x,i=t.y;this.sin_phi=Math.sin(i),this.cos_phi=Math.cos(i);var a=Vt(this.e3,this.sin_phi,this.cos_phi),h=this.a*Math.sqrt(this.c-this.ns0*a)/this.ns0,e=this.ns0*nt(s-this.long0),n=h*Math.sin(e)+this.x0,r=this.rh-h*Math.cos(e)+this.y0;return t.x=n,t.y=r,t},inverse:function(t){var s,i,a,h,e,n;return t.x-=this.x0,t.y=this.rh-t.y+this.y0,a=0<=this.ns0?(s=Math.sqrt(t.x*t.x+t.y*t.y),1):(s=-Math.sqrt(t.x*t.x+t.y*t.y),-1),(h=0)!==s&&(h=Math.atan2(a*t.x,a*t.y)),a=s*this.ns0/this.a,n=this.sphere?Math.asin((this.c-a*a)/(2*this.ns0)):(i=(this.c-a*a)/this.ns0,this.phi1z(this.e3,i)),e=nt(h/this.ns0+this.long0),t.x=e,t.y=n,t},names:["Albers_Conic_Equal_Area","Albers","aea"],phi1z:function(t,s){var i,a,h,e,n=Zt(.5*s);if(t<D)return n;for(var r=t*t,o=1;o<=25;o++)if(n+=e=.5*(h=1-(a=t*(i=Math.sin(n)))*a)*h/Math.cos(n)*(s/(1-r)-i/h+.5/t*Math.log((1-a)/(1+a))),Math.abs(e)<=1e-7)return n;return null}},ms={init:function(){this.sin_p14=Math.sin(this.lat0),this.cos_p14=Math.cos(this.lat0),this.infinity_dist=1e3*this.a,this.rc=1},forward:function(t){var s,i,a=t.x,h=t.y,e=nt(a-this.long0),n=Math.sin(h),r=Math.cos(h),o=Math.cos(e),l=0<(s=this.sin_p14*n+this.cos_p14*r*o)||Math.abs(s)<=D?(i=this.x0+this.a*r*Math.sin(e)/s,this.y0+this.a*(this.cos_p14*n-this.sin_p14*r*o)/s):(i=this.x0+this.infinity_dist*r*Math.sin(e),this.y0+this.infinity_dist*(this.cos_p14*n-this.sin_p14*r*o));return t.x=i,t.y=l,t},inverse:function(t){var s,i,a,h,e,n;return t.x=(t.x-this.x0)/this.a,t.y=(t.y-this.y0)/this.a,t.x/=this.k0,t.y/=this.k0,e=(s=Math.sqrt(t.x*t.x+t.y*t.y))?(h=Math.atan2(s,this.rc),i=Math.sin(h),a=Math.cos(h),n=Zt(a*this.sin_p14+t.y*i*this.cos_p14/s),e=Math.atan2(t.x*i,s*this.cos_p14*a-t.y*this.sin_p14*i),nt(this.long0+e)):(n=this.phic0,0),t.x=e,t.y=n,t},names:["gnom"]},ps={init:function(){this.sphere||(this.k0=ht(this.e,Math.sin(this.lat_ts),Math.cos(this.lat_ts)))},forward:function(t){var s,i,a,h=t.x,e=t.y,n=nt(h-this.long0);return a=this.sphere?(i=this.x0+this.a*n*Math.cos(this.lat_ts),this.y0+this.a*Math.sin(e)/Math.cos(this.lat_ts)):(s=Vt(this.e,Math.sin(e)),i=this.x0+this.a*this.k0*n,this.y0+this.a*s*.5/this.k0),t.x=i,t.y=a,t},inverse:function(t){var s,i;return t.x-=this.x0,t.y-=this.y0,this.sphere?(s=nt(this.long0+t.x/this.a/Math.cos(this.lat_ts)),i=Math.asin(t.y/this.a*Math.cos(this.lat_ts))):(i=function(t,s){var i=1-(1-t*t)/(2*t)*Math.log((1-t)/(1+t));if(Math.abs(Math.abs(s)-i)<1e-6)return s<0?-1*z:z;for(var a,h,e,n,r=Math.asin(.5*s),o=0;o<30;o++)if(h=Math.sin(r),e=Math.cos(r),n=t*h,r+=a=Math.pow(1-n*n,2)/(2*e)*(s/(1-t*t)-h/(1-n*n)+.5/t*Math.log((1-n)/(1+n))),Math.abs(a)<=1e-10)return r;return NaN}(this.e,2*t.y*this.k0/this.a),s=nt(this.long0+t.x/(this.a*this.k0))),t.x=s,t.y=i,t},names:["cea"]},ds={init:function(){this.x0=this.x0||0,this.y0=this.y0||0,this.lat0=this.lat0||0,this.long0=this.long0||0,this.lat_ts=this.lat_ts||0,this.title=this.title||"Equidistant Cylindrical (Plate Carre)",this.rc=Math.cos(this.lat_ts)},forward:function(t){var s=t.x,i=t.y,a=nt(s-this.long0),h=Jt(i-this.lat0);return t.x=this.x0+this.a*a*this.rc,t.y=this.y0+this.a*h,t},inverse:function(t){var s=t.x,i=t.y;return t.x=nt(this.long0+(s-this.x0)/(this.a*this.rc)),t.y=Jt(this.lat0+(i-this.y0)/this.a),t},names:["Equirectangular","Equidistant_Cylindrical","eqc"]},ys={init:function(){this.temp=this.b/this.a,this.es=1-Math.pow(this.temp,2),this.e=Math.sqrt(this.es),this.e0=Ft(this.es),this.e1=Qt(this.es),this.e2=Wt(this.es),this.e3=Xt(this.es),this.ml0=this.a*Ut(this.e0,this.e1,this.e2,this.e3,this.lat0)},forward:function(t){var s,i,a,h=t.x,e=t.y,n=nt(h-this.long0),r=n*Math.sin(e);return a=this.sphere?Math.abs(e)<=D?(i=this.a*n,-1*this.a*this.lat0):(i=this.a*Math.sin(r)/Math.tan(e),this.a*(Jt(e-this.lat0)+(1-Math.cos(r))/Math.tan(e))):Math.abs(e)<=D?(i=this.a*n,-1*this.ml0):(i=(s=Ht(this.a,this.e,Math.sin(e))/Math.tan(e))*Math.sin(r),this.a*Ut(this.e0,this.e1,this.e2,this.e3,e)-this.ml0+s*(1-Math.cos(r))),t.x=i+this.x0,t.y=a+this.y0,t},inverse:function(t){var s,i,a,h,e,n,r,o,l=t.x-this.x0,M=t.y-this.y0;if(this.sphere)if(Math.abs(M+this.a*this.lat0)<=D)s=nt(l/this.a+this.long0),i=0;else{for(var c,u=this.lat0+M/this.a,f=l*l/this.a/this.a+u*u,m=u,p=20;p;--p)if(m+=a=-1*(u*(m*(c=Math.tan(m))+1)-m-.5*(m*m+f)*c)/((m-u)/c-1),Math.abs(a)<=D){i=m;break}s=nt(this.long0+Math.asin(l*Math.tan(m)/this.a)/Math.sin(i))}else if(Math.abs(M+this.ml0)<=D)i=0,s=nt(this.long0+l/this.a);else{for(u=(this.ml0+M)/this.a,f=l*l/this.a/this.a+u*u,m=u,p=20;p;--p)if(o=this.e*Math.sin(m),h=Math.sqrt(1-o*o)*Math.tan(m),e=this.a*Ut(this.e0,this.e1,this.e2,this.e3,m),n=this.e0-2*this.e1*Math.cos(2*m)+4*this.e2*Math.cos(4*m)-6*this.e3*Math.cos(6*m),m-=a=(u*(h*(r=e/this.a)+1)-r-.5*h*(r*r+f))/(this.es*Math.sin(2*m)*(r*r+f-2*u*r)/(4*h)+(u-r)*(h*n-2/Math.sin(2*m))-n),Math.abs(a)<=D){i=m;break}h=Math.sqrt(1-this.es*Math.pow(Math.sin(i),2))*Math.tan(i),s=nt(this.long0+Math.asin(l*h/this.a)/Math.sin(i))}return t.x=s,t.y=i,t},names:["Polyconic","poly"]},_s={init:function(){this.A=[],this.A[1]=.6399175073,this.A[2]=-.1358797613,this.A[3]=.063294409,this.A[4]=-.02526853,this.A[5]=.0117879,this.A[6]=-.0055161,this.A[7]=.0026906,this.A[8]=-.001333,this.A[9]=67e-5,this.A[10]=-34e-5,this.B_re=[],this.B_im=[],this.B_re[1]=.7557853228,this.B_im[1]=0,this.B_re[2]=.249204646,this.B_im[2]=.003371507,this.B_re[3]=-.001541739,this.B_im[3]=.04105856,this.B_re[4]=-.10162907,this.B_im[4]=.01727609,this.B_re[5]=-.26623489,this.B_im[5]=-.36249218,this.B_re[6]=-.6870983,this.B_im[6]=-1.1651967,this.C_re=[],this.C_im=[],this.C_re[1]=1.3231270439,this.C_im[1]=0,this.C_re[2]=-.577245789,this.C_im[2]=-.007809598,this.C_re[3]=.508307513,this.C_im[3]=-.112208952,this.C_re[4]=-.15094762,this.C_im[4]=.18200602,this.C_re[5]=1.01418179,this.C_im[5]=1.64497696,this.C_re[6]=1.9660549,this.C_im[6]=2.5127645,this.D=[],this.D[1]=1.5627014243,this.D[2]=.5185406398,this.D[3]=-.03333098,this.D[4]=-.1052906,this.D[5]=-.0368594,this.D[6]=.007317,this.D[7]=.0122,this.D[8]=.00394,this.D[9]=-.0013},forward:function(t){for(var s=t.x,i=t.y-this.lat0,a=s-this.long0,h=i/j*1e-5,e=a,n=1,r=0,o=1;o<=10;o++)n*=h,r+=this.A[o]*n;var l,M=r,c=e,u=1,f=0,m=0,p=0;for(o=1;o<=6;o++)l=f*M+u*c,u=u*M-f*c,f=l,m=m+this.B_re[o]*u-this.B_im[o]*f,p=p+this.B_im[o]*u+this.B_re[o]*f;return t.x=p*this.a+this.x0,t.y=m*this.a+this.y0,t},inverse:function(t){var s,i=t.x,a=t.y,h=i-this.x0,e=(a-this.y0)/this.a,n=h/this.a,r=1,o=0,l=0,M=0;for(y=1;y<=6;y++)s=o*e+r*n,r=r*e-o*n,o=s,l=l+this.C_re[y]*r-this.C_im[y]*o,M=M+this.C_im[y]*r+this.C_re[y]*o;for(var c=0;c<this.iterations;c++){for(var u,f=l,m=M,p=e,d=n,y=2;y<=6;y++)u=m*l+f*M,f=f*l-m*M,m=u,p+=(y-1)*(this.B_re[y]*f-this.B_im[y]*m),d+=(y-1)*(this.B_im[y]*f+this.B_re[y]*m);f=1,m=0;var _=this.B_re[1],x=this.B_im[1];for(y=2;y<=6;y++)u=m*l+f*M,f=f*l-m*M,m=u,_+=y*(this.B_re[y]*f-this.B_im[y]*m),x+=y*(this.B_im[y]*f+this.B_re[y]*m);var g=_*_+x*x,l=(p*_+d*x)/g,M=(d*_-p*x)/g}var b=l,v=M,w=1,C=0;for(y=1;y<=9;y++)w*=b,C+=this.D[y]*w;var P=this.lat0+C*j*1e5,S=this.long0+v;return t.x=S,t.y=P,t},names:["New_Zealand_Map_Grid","nzmg"]},xs={init:function(){},forward:function(t){var s=t.x,i=t.y,a=nt(s-this.long0),h=this.x0+this.a*a,e=this.y0+this.a*Math.log(Math.tan(Math.PI/4+i/2.5))*1.25;return t.x=h,t.y=e,t},inverse:function(t){t.x-=this.x0,t.y-=this.y0;var s=nt(this.long0+t.x/this.a),i=2.5*(Math.atan(Math.exp(.8*t.y/this.a))-Math.PI/4);return t.x=s,t.y=i,t},names:["Miller_Cylindrical","mill"]},gs={init:function(){this.sphere?(this.n=1,this.m=0,this.es=0,this.C_y=Math.sqrt((this.m+1)/this.n),this.C_x=this.C_y/(this.m+1)):this.en=At(this.es)},forward:function(t){var s=t.x,i=t.y,s=nt(s-this.long0);if(this.sphere){if(this.m)for(var a=this.n*Math.sin(i),h=20;h;--h){var e=(this.m*i+Math.sin(i)-a)/(this.m+Math.cos(i));if(i-=e,Math.abs(e)<D)break}else i=1!==this.n?Math.asin(this.n*Math.sin(i)):i;l=this.a*this.C_x*s*(this.m+Math.cos(i)),o=this.a*this.C_y*i}else var n=Math.sin(i),r=Math.cos(i),o=this.a*Gt(i,n,r,this.en),l=this.a*s*r/Math.sqrt(1-this.es*n*n);return t.x=l,t.y=o,t},inverse:function(t){var s,i,a,h;return t.x-=this.x0,a=t.x/this.a,t.y-=this.y0,s=t.y/this.a,this.sphere?(s/=this.C_y,a/=this.C_x*(this.m+Math.cos(s)),this.m?s=Zt((this.m*s+Math.sin(s))/this.n):1!==this.n&&(s=Zt(Math.sin(s)/this.n)),a=nt(a+this.long0),s=Jt(s)):(s=jt(t.y/this.a,this.es,this.en),(h=Math.abs(s))<z?(h=Math.sin(s),i=this.long0+t.x*Math.sqrt(1-this.es*h*h)/(this.a*Math.cos(s)),a=nt(i)):h-D<z&&(a=this.long0)),t.x=a,t.y=s,t},names:["Sinusoidal","sinu"]},bs={init:function(){},forward:function(t){for(var s=t.x,i=t.y,a=nt(s-this.long0),h=i,e=Math.PI*Math.sin(i);;){var n=-(h+Math.sin(h)-e)/(1+Math.cos(h));if(h+=n,Math.abs(n)<D)break}h/=2,Math.PI/2-Math.abs(i)<D&&(a=0);var r=.900316316158*this.a*a*Math.cos(h)+this.x0,o=1.4142135623731*this.a*Math.sin(h)+this.y0;return t.x=r,t.y=o,t},inverse:function(t){var s,i;t.x-=this.x0,t.y-=this.y0,i=t.y/(1.4142135623731*this.a),.999999999999<Math.abs(i)&&(i=.999999999999),s=Math.asin(i);var a=nt(this.long0+t.x/(.900316316158*this.a*Math.cos(s)));a<-Math.PI&&(a=-Math.PI),a>Math.PI&&(a=Math.PI),i=(2*s+Math.sin(2*s))/Math.PI,1<Math.abs(i)&&(i=1);var h=Math.asin(i);return t.x=a,t.y=h,t},names:["Mollweide","moll"]},vs={init:function(){Math.abs(this.lat1+this.lat2)<D||(this.lat2=this.lat2||this.lat1,this.temp=this.b/this.a,this.es=1-Math.pow(this.temp,2),this.e=Math.sqrt(this.es),this.e0=Ft(this.es),this.e1=Qt(this.es),this.e2=Wt(this.es),this.e3=Xt(this.es),this.sinphi=Math.sin(this.lat1),this.cosphi=Math.cos(this.lat1),this.ms1=ht(this.e,this.sinphi,this.cosphi),this.ml1=Ut(this.e0,this.e1,this.e2,this.e3,this.lat1),Math.abs(this.lat1-this.lat2)<D?this.ns=this.sinphi:(this.sinphi=Math.sin(this.lat2),this.cosphi=Math.cos(this.lat2),this.ms2=ht(this.e,this.sinphi,this.cosphi),this.ml2=Ut(this.e0,this.e1,this.e2,this.e3,this.lat2),this.ns=(this.ms1-this.ms2)/(this.ml2-this.ml1)),this.g=this.ml1+this.ms1/this.ns,this.ml0=Ut(this.e0,this.e1,this.e2,this.e3,this.lat0),this.rh=this.a*(this.g-this.ml0))},forward:function(t){var s,i,a=t.x,h=t.y;i=this.sphere?this.a*(this.g-h):(s=Ut(this.e0,this.e1,this.e2,this.e3,h),this.a*(this.g-s));var e=this.ns*nt(a-this.long0),n=this.x0+i*Math.sin(e),r=this.y0+this.rh-i*Math.cos(e);return t.x=n,t.y=r,t},inverse:function(t){var s,i;t.x-=this.x0,t.y=this.rh-t.y+this.y0,s=0<=this.ns?(i=Math.sqrt(t.x*t.x+t.y*t.y),1):(i=-Math.sqrt(t.x*t.x+t.y*t.y),-1);var a=0;if(0!==i&&(a=Math.atan2(s*t.x,s*t.y)),this.sphere)return n=nt(this.long0+a/this.ns),e=Jt(this.g-i/this.a),t.x=n,t.y=e,t;var h=this.g-i/this.a,e=Kt(h,this.e0,this.e1,this.e2,this.e3),n=nt(this.long0+a/this.ns);return t.x=n,t.y=e,t},names:["Equidistant_Conic","eqdc"]},ws={init:function(){this.R=this.a},forward:function(t){var s,i=t.x,a=t.y,h=nt(i-this.long0);Math.abs(a)<=D&&(s=this.x0+this.R*h,d=this.y0);var e=Zt(2*Math.abs(a/Math.PI));(Math.abs(h)<=D||Math.abs(Math.abs(a)-z)<=D)&&(s=this.x0,d=0<=a?this.y0+Math.PI*this.R*Math.tan(.5*e):this.y0+Math.PI*this.R*-Math.tan(.5*e));var n=.5*Math.abs(Math.PI/h-h/Math.PI),r=n*n,o=Math.sin(e),l=Math.cos(e),M=l/(o+l-1),c=M*M,u=M*(2/o-1),f=u*u,m=Math.PI*this.R*(n*(M-f)+Math.sqrt(r*(M-f)*(M-f)-(f+r)*(c-f)))/(f+r);h<0&&(m=-m),s=this.x0+m;var p=r+M,m=Math.PI*this.R*(u*p-n*Math.sqrt((f+r)*(1+r)-p*p))/(f+r),d=0<=a?this.y0+m:this.y0-m;return t.x=s,t.y=d,t},inverse:function(t){var s,i,a,h,e,n,r,o,l,M,c,u;return t.x-=this.x0,t.y-=this.y0,c=Math.PI*this.R,e=(a=t.x/c)*a+(h=t.y/c)*h,c=3*(h*h/(o=-2*(n=-Math.abs(h)*(1+e))+1+2*h*h+e*e)+(2*(r=n-2*h*h+a*a)*r*r/o/o/o-9*n*r/o/o)/27)/(l=(n-r*r/3/o)/o)/(M=2*Math.sqrt(-l/3)),1<Math.abs(c)&&(c=0<=c?1:-1),u=Math.acos(c)/3,i=0<=t.y?(-M*Math.cos(u+Math.PI/3)-r/3/o)*Math.PI:-(-M*Math.cos(u+Math.PI/3)-r/3/o)*Math.PI,s=Math.abs(a)<D?this.long0:nt(this.long0+Math.PI*(e-1+Math.sqrt(1+2*(a*a-h*h)+e*e))/2/a),t.x=s,t.y=i,t},names:["Van_der_Grinten_I","VanDerGrinten","vandg"]},Cs={init:function(){this.sin_p12=Math.sin(this.lat0),this.cos_p12=Math.cos(this.lat0)},forward:function(t){var s,i,a,h,e,n,r,o,l,M,c,u,f,m,p,d,y,_,x,g,b,v,w=t.x,C=t.y,P=Math.sin(t.y),S=Math.cos(t.y),N=nt(w-this.long0);return this.sphere?Math.abs(this.sin_p12-1)<=D?(t.x=this.x0+this.a*(z-C)*Math.sin(N),t.y=this.y0-this.a*(z-C)*Math.cos(N)):Math.abs(this.sin_p12+1)<=D?(t.x=this.x0+this.a*(z+C)*Math.sin(N),t.y=this.y0+this.a*(z+C)*Math.cos(N)):(_=this.sin_p12*P+this.cos_p12*S*Math.cos(N),y=(d=Math.acos(_))?d/Math.sin(d):1,t.x=this.x0+this.a*y*S*Math.sin(N),t.y=this.y0+this.a*y*(this.cos_p12*P-this.sin_p12*S*Math.cos(N))):(s=Ft(this.es),i=Qt(this.es),a=Wt(this.es),h=Xt(this.es),Math.abs(this.sin_p12-1)<=D?(e=this.a*Ut(s,i,a,h,z),n=this.a*Ut(s,i,a,h,C),t.x=this.x0+(e-n)*Math.sin(N),t.y=this.y0-(e-n)*Math.cos(N)):Math.abs(this.sin_p12+1)<=D?(e=this.a*Ut(s,i,a,h,z),n=this.a*Ut(s,i,a,h,C),t.x=this.x0+(e+n)*Math.sin(N),t.y=this.y0+(e+n)*Math.cos(N)):(r=P/S,o=Ht(this.a,this.e,this.sin_p12),l=Ht(this.a,this.e,P),M=Math.atan((1-this.es)*r+this.es*o*this.sin_p12/(l*S)),x=0===(c=Math.atan2(Math.sin(N),this.cos_p12*Math.tan(M)-this.sin_p12*Math.cos(N)))?Math.asin(this.cos_p12*Math.sin(M)-this.sin_p12*Math.cos(M)):Math.abs(Math.abs(c)-Math.PI)<=D?-Math.asin(this.cos_p12*Math.sin(M)-this.sin_p12*Math.cos(M)):Math.asin(Math.sin(N)*Math.cos(M)/Math.sin(c)),u=this.e*this.sin_p12/Math.sqrt(1-this.es),d=o*x*(1-(g=x*x)*(p=(f=this.e*this.cos_p12*Math.cos(c)/Math.sqrt(1-this.es))*f)*(1-p)/6+(b=g*x)/8*(m=u*f)*(1-2*p)+(v=b*x)/120*(p*(4-7*p)-3*u*u*(1-7*p))-v*x/48*m),t.x=this.x0+d*Math.sin(c),t.y=this.y0+d*Math.cos(c))),t},inverse:function(t){var s,i,a,h,e,n,r,o,l,M,c,u,f,m,p,d,y,_,x,g,b,v,w;if(t.x-=this.x0,t.y-=this.y0,this.sphere){if((s=Math.sqrt(t.x*t.x+t.y*t.y))>2*z*this.a)return;return i=s/this.a,a=Math.sin(i),h=Math.cos(i),e=this.long0,Math.abs(s)<=D?n=this.lat0:(n=Zt(h*this.sin_p12+t.y*a*this.cos_p12/s),r=Math.abs(this.lat0)-z,e=nt(Math.abs(r)<=D?0<=this.lat0?this.long0+Math.atan2(t.x,-t.y):this.long0-Math.atan2(-t.x,t.y):this.long0+Math.atan2(t.x*a,s*this.cos_p12*h-t.y*this.sin_p12*a))),t.x=e,t.y=n,t}return o=Ft(this.es),l=Qt(this.es),M=Wt(this.es),c=Xt(this.es),Math.abs(this.sin_p12-1)<=D?(u=this.a*Ut(o,l,M,c,z),s=Math.sqrt(t.x*t.x+t.y*t.y),n=Kt((u-s)/this.a,o,l,M,c),e=nt(this.long0+Math.atan2(t.x,-1*t.y))):Math.abs(this.sin_p12+1)<=D?(u=this.a*Ut(o,l,M,c,z),s=Math.sqrt(t.x*t.x+t.y*t.y),n=Kt((s-u)/this.a,o,l,M,c),e=nt(this.long0+Math.atan2(t.x,t.y))):(s=Math.sqrt(t.x*t.x+t.y*t.y),p=Math.atan2(t.x,t.y),f=Ht(this.a,this.e,this.sin_p12),d=Math.cos(p),_=-(y=this.e*this.cos_p12*d)*y/(1-this.es),x=3*this.es*(1-_)*this.sin_p12*this.cos_p12*d/(1-this.es),v=1-_*(b=(g=s/f)-_*(1+_)*Math.pow(g,3)/6-x*(1+3*_)*Math.pow(g,4)/24)*b/2-g*b*b*b/6,m=Math.asin(this.sin_p12*Math.cos(b)+this.cos_p12*Math.sin(b)*d),e=nt(this.long0+Math.asin(Math.sin(p)*Math.sin(b)/Math.cos(m))),w=Math.sin(m),n=Math.atan2((w-this.es*v*this.sin_p12)*Math.tan(m),w*(1-this.es))),t.x=e,t.y=n,t},names:["Azimuthal_Equidistant","aeqd"]},Ps={init:function(){this.sin_p14=Math.sin(this.lat0),this.cos_p14=Math.cos(this.lat0)},forward:function(t){var s,i,a,h=t.x,e=t.y,n=nt(h-this.long0),r=Math.sin(e),o=Math.cos(e),l=Math.cos(n);return(0<(s=this.sin_p14*r+this.cos_p14*o*l)||Math.abs(s)<=D)&&(i=this.a*o*Math.sin(n),a=this.y0+this.a*(this.cos_p14*r-this.sin_p14*o*l)),t.x=i,t.y=a,t},inverse:function(t){var s,i,a,h,e,n,r;return t.x-=this.x0,t.y-=this.y0,s=Math.sqrt(t.x*t.x+t.y*t.y),i=Zt(s/this.a),a=Math.sin(i),h=Math.cos(i),n=this.long0,Math.abs(s)<=D?r=this.lat0:(r=Zt(h*this.sin_p14+t.y*a*this.cos_p14/s),e=Math.abs(this.lat0)-z,n=Math.abs(e)<=D?nt(0<=this.lat0?this.long0+Math.atan2(t.x,-t.y):this.long0-Math.atan2(-t.x,t.y)):nt(this.long0+Math.atan2(t.x*a,s*this.cos_p14*h-t.y*this.sin_p14*a))),t.x=n,t.y=r,t},names:["ortho"]},Ss=1,Ns=2,ks=3,Es=4,qs=5,Is=6,Os=1,As=2,Gs=3,js=4,zs={init:function(){this.x0=this.x0||0,this.y0=this.y0||0,this.lat0=this.lat0||0,this.long0=this.long0||0,this.lat_ts=this.lat_ts||0,this.title=this.title||"Quadrilateralized Spherical Cube",this.lat0>=z-U/2?this.face=qs:this.lat0<=-(z-U/2)?this.face=Is:Math.abs(this.long0)<=U?this.face=Ss:Math.abs(this.long0)<=z+U?this.face=0<this.long0?Ns:Es:this.face=ks,0!==this.es&&(this.one_minus_f=1-(this.a-this.b)/this.a,this.one_minus_f_squared=this.one_minus_f*this.one_minus_f)},forward:function(t){var s,i,a,h,e,n,r,o,l,M,c,u,f={x:0,y:0},m={value:0};return t.x-=this.long0,s=0!==this.es?Math.atan(this.one_minus_f_squared*Math.tan(t.y)):t.y,i=t.x,this.face===qs?(h=z-s,a=U<=i&&i<=z+U?(m.value=Os,i-z):z+U<i||i<=-(z+U)?(m.value=As,0<i?i-Q:i+Q):-(z+U)<i&&i<=-U?(m.value=Gs,i+z):(m.value=js,i)):this.face===Is?(h=z+s,a=U<=i&&i<=z+U?(m.value=Os,z-i):i<U&&-U<=i?(m.value=As,-i):i<-U&&-(z+U)<=i?(m.value=Gs,-i-z):(m.value=js,0<i?Q-i:-i-Q)):(this.face===Ns?i=S(i,+z):this.face===ks?i=S(i,+Q):this.face===Es&&(i=S(i,-z)),M=Math.sin(s),c=Math.cos(s),u=Math.sin(i),r=c*Math.cos(i),o=c*u,l=M,this.face===Ss?a=P(h=Math.acos(r),l,o,m):this.face===Ns?a=P(h=Math.acos(o),l,-r,m):this.face===ks?a=P(h=Math.acos(-r),l,-o,m):this.face===Es?a=P(h=Math.acos(-o),l,r,m):(h=a=0,m.value=Os)),n=Math.atan(12/Q*(a+Math.acos(Math.sin(a)*Math.cos(U))-z)),e=Math.sqrt((1-Math.cos(h))/(Math.cos(n)*Math.cos(n))/(1-Math.cos(Math.atan(1/Math.cos(a))))),m.value===As?n+=z:m.value===Gs?n+=Q:m.value===js&&(n+=1.5*Q),f.x=e*Math.cos(n),f.y=e*Math.sin(n),f.x=f.x*this.a+this.x0,f.y=f.y*this.a+this.y0,t.x=f.x,t.y=f.y,t},inverse:function(t){var s,i,a,h,e,n,r,o,l,M,c,u,f,m,p,d={lam:0,phi:0},y={value:0};return t.x=(t.x-this.x0)/this.a,t.y=(t.y-this.y0)/this.a,i=Math.atan(Math.sqrt(t.x*t.x+t.y*t.y)),s=Math.atan2(t.y,t.x),0<=t.x&&t.x>=Math.abs(t.y)?y.value=Os:0<=t.y&&t.y>=Math.abs(t.x)?(y.value=As,s-=z):t.x<0&&-t.x>=Math.abs(t.y)?(y.value=Gs,s=s<0?s+Q:s-Q):(y.value=js,s+=z),c=Q/12*Math.tan(s),e=Math.sin(c)/(Math.cos(c)-1/Math.sqrt(2)),n=Math.atan(e),(r=1-(a=Math.cos(s))*a*(h=Math.tan(i))*h*(1-Math.cos(Math.atan(1/Math.cos(n)))))<-1?r=-1:1<r&&(r=1),this.face===qs?(o=Math.acos(r),d.phi=z-o,y.value===Os?d.lam=n+z:y.value===As?d.lam=n<0?n+Q:n-Q:y.value===Gs?d.lam=n-z:d.lam=n):this.face===Is?(o=Math.acos(r),d.phi=o-z,y.value===Os?d.lam=z-n:y.value===As?d.lam=-n:y.value===Gs?d.lam=-n-z:d.lam=n<0?-n-Q:Q-n):(c=(l=r)*l,u=1<=(c+=(M=1<=c?0:Math.sqrt(1-c)*Math.sin(n))*M)?0:Math.sqrt(1-c),y.value===As?(c=u,u=-M,M=c):y.value===Gs?(u=-u,M=-M):y.value===js&&(c=u,u=M,M=-c),this.face===Ns?(c=l,l=-u,u=c):this.face===ks?(l=-l,u=-u):this.face===Es&&(c=l,l=u,u=-c),d.phi=Math.acos(-M)-z,d.lam=Math.atan2(u,l),this.face===Ns?d.lam=S(d.lam,-z):this.face===ks?d.lam=S(d.lam,-Q):this.face===Es&&(d.lam=S(d.lam,+z))),0!==this.es&&(f=d.phi<0?1:0,m=Math.tan(d.phi),p=this.b/Math.sqrt(m*m+this.one_minus_f_squared),d.phi=Math.atan(Math.sqrt(this.a*this.a-p*p)/(this.one_minus_f*p)),f&&(d.phi=-d.phi)),d.lam+=this.long0,t.x=d.lam,t.y=d.phi,t},names:["Quadrilateralized Spherical Cube","Quadrilateralized_Spherical_Cube","qsc"]},Rs=[[1,22199e-21,-715515e-10,31103e-10],[.9986,-482243e-9,-24897e-9,-13309e-10],[.9954,-83103e-8,-448605e-10,-9.86701e-7],[.99,-.00135364,-59661e-9,36777e-10],[.9822,-.00167442,-449547e-11,-572411e-11],[.973,-.00214868,-903571e-10,1.8736e-8],[.96,-.00305085,-900761e-10,164917e-11],[.9427,-.00382792,-653386e-10,-26154e-10],[.9216,-.00467746,-10457e-8,481243e-11],[.8962,-.00536223,-323831e-10,-543432e-11],[.8679,-.00609363,-113898e-9,332484e-11],[.835,-.00698325,-640253e-10,9.34959e-7],[.7986,-.00755338,-500009e-10,9.35324e-7],[.7597,-.00798324,-35971e-9,-227626e-11],[.7186,-.00851367,-701149e-10,-86303e-10],[.6732,-.00986209,-199569e-9,191974e-10],[.6213,-.010418,883923e-10,624051e-11],[.5722,-.00906601,182e-6,624051e-11],[.5322,-.00677797,275608e-9,624051e-11]],Ls=[[-520417e-23,.0124,121431e-23,-845284e-16],[.062,.0124,-1.26793e-9,4.22642e-10],[.124,.0124,5.07171e-9,-1.60604e-9],[.186,.0123999,-1.90189e-8,6.00152e-9],[.248,.0124002,7.10039e-8,-2.24e-8],[.31,.0123992,-2.64997e-7,8.35986e-8],[.372,.0124029,9.88983e-7,-3.11994e-7],[.434,.0123893,-369093e-11,-4.35621e-7],[.4958,.0123198,-102252e-10,-3.45523e-7],[.5571,.0121916,-154081e-10,-5.82288e-7],[.6176,.0119938,-241424e-10,-5.25327e-7],[.6769,.011713,-320223e-10,-5.16405e-7],[.7346,.0113541,-397684e-10,-6.09052e-7],[.7903,.0109107,-489042e-10,-104739e-11],[.8435,.0103431,-64615e-9,-1.40374e-9],[.8936,.00969686,-64636e-9,-8547e-9],[.9394,.00840947,-192841e-9,-42106e-10],[.9761,.00616527,-256e-6,-42106e-10],[1,.00328947,-319159e-9,-42106e-10]],Ts=B/5,Ds=1/Ts,Bs={init:function(){this.x0=this.x0||0,this.y0=this.y0||0,this.long0=this.long0||0,this.es=0,this.title=this.title||"Robinson"},forward:function(t){var s=nt(t.x-this.long0),i=Math.abs(t.y),a=Math.floor(i*Ts);a<0?a=0:18<=a&&(a=17);var h={x:Yt(Rs[a],i=B*(i-Ds*a))*s,y:Yt(Ls[a],i)};return t.y<0&&(h.y=-h.y),h.x=h.x*this.a*.8487+this.x0,h.y=h.y*this.a*1.3523+this.y0,h},inverse:function(t){var a={x:(t.x-this.x0)/(.8487*this.a),y:Math.abs(t.y-this.y0)/(1.3523*this.a)};if(1<=a.y)a.x/=Rs[18][0],a.y=t.y<0?-z:z;else{var s=Math.floor(18*a.y);for(s<0?s=0:18<=s&&(s=17);;)if(Ls[s][0]>a.y)--s;else{if(!(Ls[s+1][0]<=a.y))break;++s}var h=Ls[s],i=function(t,s,i,a){for(var h=s;a;--a){var e=t(h);if(h-=e,Math.abs(e)<i)break}return h}(function(t){return(Yt(h,t)-a.y)/(i=t,(s=h)[1]+i*(2*s[2]+3*i*s[3]));var s,i},i=5*(a.y-h[0])/(Ls[s+1][0]-h[0]),D,100);a.x/=Yt(Rs[s],i),a.y=(5*s+i)*N,t.y<0&&(a.y=-a.y)}return a.x=nt(a.x+this.long0),a},names:["Robinson","robin"]},Us={init:function(){this.name="geocent"},forward:function(t){return M(t,this.es,this.a)},inverse:function(t){return c(t,this.es,this.a,this.b)},names:["Geocentric","geocentric","geocent","Geocent"]};return a.defaultDatum="WGS84",a.Proj=q,a.WGS84=new a.Proj("WGS84"),a.Point=C,a.toPoint=bt,a.defs=l,a.transform=f,a.mgrs=Ot,a.version="2.6.2",($t=a).Proj.projections.add(ss),$t.Proj.projections.add(is),$t.Proj.projections.add(as),$t.Proj.projections.add(es),$t.Proj.projections.add(ns),$t.Proj.projections.add(rs),$t.Proj.projections.add(os),$t.Proj.projections.add(ls),$t.Proj.projections.add(Ms),$t.Proj.projections.add(cs),$t.Proj.projections.add(us),$t.Proj.projections.add(fs),$t.Proj.projections.add(ms),$t.Proj.projections.add(ps),$t.Proj.projections.add(ds),$t.Proj.projections.add(ys),$t.Proj.projections.add(_s),$t.Proj.projections.add(xs),$t.Proj.projections.add(gs),$t.Proj.projections.add(bs),$t.Proj.projections.add(vs),$t.Proj.projections.add(ws),$t.Proj.projections.add(Cs),$t.Proj.projections.add(Ps),$t.Proj.projections.add(zs),$t.Proj.projections.add(Bs),$t.Proj.projections.add(Us),a});</script>
<script>(function (factory) {
	var L, proj4;
	if (typeof define === 'function' && define.amd) {
		// AMD
		define(['leaflet', 'proj4'], factory);
	} else if (typeof module === 'object' && typeof module.exports === "object") {
		// Node/CommonJS
		L = require('leaflet');
		proj4 = require('proj4');
		module.exports = factory(L, proj4);
	} else {
		// Browser globals
		if (typeof window.L === 'undefined' || typeof window.proj4 === 'undefined')
			throw 'Leaflet and proj4 must be loaded first';
		factory(window.L, window.proj4);
	}
}(function (L, proj4) {
	if (proj4.__esModule && proj4.default) {
		// If proj4 was bundled as an ES6 module, unwrap it to get
		// to the actual main proj4 object.
		// See discussion in https://github.com/kartena/Proj4Leaflet/pull/147
		proj4 = proj4.default;
	}
 
	L.Proj = {};

	L.Proj._isProj4Obj = function(a) {
		return (typeof a.inverse !== 'undefined' &&
			typeof a.forward !== 'undefined');
	};

	L.Proj.Projection = L.Class.extend({
		initialize: function(code, def, bounds) {
			var isP4 = L.Proj._isProj4Obj(code);
			this._proj = isP4 ? code : this._projFromCodeDef(code, def);
			this.bounds = isP4 ? def : bounds;
		},

		project: function (latlng) {
			var point = this._proj.forward([latlng.lng, latlng.lat]);
			return new L.Point(point[0], point[1]);
		},

		unproject: function (point, unbounded) {
			var point2 = this._proj.inverse([point.x, point.y]);
			return new L.LatLng(point2[1], point2[0], unbounded);
		},

		_projFromCodeDef: function(code, def) {
			if (def) {
				proj4.defs(code, def);
			} else if (proj4.defs[code] === undefined) {
				var urn = code.split(':');
				if (urn.length > 3) {
					code = urn[urn.length - 3] + ':' + urn[urn.length - 1];
				}
				if (proj4.defs[code] === undefined) {
					throw 'No projection definition for code ' + code;
				}
			}

			return proj4(code);
		}
	});

	L.Proj.CRS = L.Class.extend({
		includes: L.CRS,

		options: {
			transformation: new L.Transformation(1, 0, -1, 0)
		},

		initialize: function(a, b, c) {
			var code,
			    proj,
			    def,
			    options;

			if (L.Proj._isProj4Obj(a)) {
				proj = a;
				code = proj.srsCode;
				options = b || {};

				this.projection = new L.Proj.Projection(proj, options.bounds);
			} else {
				code = a;
				def = b;
				options = c || {};
				this.projection = new L.Proj.Projection(code, def, options.bounds);
			}

			L.Util.setOptions(this, options);
			this.code = code;
			this.transformation = this.options.transformation;

			if (this.options.origin) {
				this.transformation =
					new L.Transformation(1, -this.options.origin[0],
						-1, this.options.origin[1]);
			}

			if (this.options.scales) {
				this._scales = this.options.scales;
			} else if (this.options.resolutions) {
				this._scales = [];
				for (var i = this.options.resolutions.length - 1; i >= 0; i--) {
					if (this.options.resolutions[i]) {
						this._scales[i] = 1 / this.options.resolutions[i];
					}
				}
			}

			this.infinite = !this.options.bounds;

		},

		scale: function(zoom) {
			var iZoom = Math.floor(zoom),
				baseScale,
				nextScale,
				scaleDiff,
				zDiff;
			if (zoom === iZoom) {
				return this._scales[zoom];
			} else {
				// Non-integer zoom, interpolate
				baseScale = this._scales[iZoom];
				nextScale = this._scales[iZoom + 1];
				scaleDiff = nextScale - baseScale;
				zDiff = (zoom - iZoom);
				return baseScale + scaleDiff * zDiff;
			}
		},

		zoom: function(scale) {
			// Find closest number in this._scales, down
			var downScale = this._closestElement(this._scales, scale),
				downZoom = this._scales.indexOf(downScale),
				nextScale,
				nextZoom,
				scaleDiff;
			// Check if scale is downScale => return array index
			if (scale === downScale) {
				return downZoom;
			}
			if (downScale === undefined) {
				return -Infinity;
			}
			// Interpolate
			nextZoom = downZoom + 1;
			nextScale = this._scales[nextZoom];
			if (nextScale === undefined) {
				return Infinity;
			}
			scaleDiff = nextScale - downScale;
			return (scale - downScale) / scaleDiff + downZoom;
		},

		distance: L.CRS.Earth.distance,

		R: L.CRS.Earth.R,

		/* Get the closest lowest element in an array */
		_closestElement: function(array, element) {
			var low;
			for (var i = array.length; i--;) {
				if (array[i] <= element && (low === undefined || low < array[i])) {
					low = array[i];
				}
			}
			return low;
		}
	});

	L.Proj.GeoJSON = L.GeoJSON.extend({
		initialize: function(geojson, options) {
			this._callLevel = 0;
			L.GeoJSON.prototype.initialize.call(this, geojson, options);
		},

		addData: function(geojson) {
			var crs;

			if (geojson) {
				if (geojson.crs && geojson.crs.type === 'name') {
					crs = new L.Proj.CRS(geojson.crs.properties.name);
				} else if (geojson.crs && geojson.crs.type) {
					crs = new L.Proj.CRS(geojson.crs.type + ':' + geojson.crs.properties.code);
				}

				if (crs !== undefined) {
					this.options.coordsToLatLng = function(coords) {
						var point = L.point(coords[0], coords[1]);
						return crs.projection.unproject(point);
					};
				}
			}

			// Base class' addData might call us recursively, but
			// CRS shouldn't be cleared in that case, since CRS applies
			// to the whole GeoJSON, inluding sub-features.
			this._callLevel++;
			try {
				L.GeoJSON.prototype.addData.call(this, geojson);
			} finally {
				this._callLevel--;
				if (this._callLevel === 0) {
					delete this.options.coordsToLatLng;
				}
			}
		}
	});

	L.Proj.geoJson = function(geojson, options) {
		return new L.Proj.GeoJSON(geojson, options);
	};

	L.Proj.ImageOverlay = L.ImageOverlay.extend({
		initialize: function (url, bounds, options) {
			L.ImageOverlay.prototype.initialize.call(this, url, null, options);
			this._projectedBounds = bounds;
		},

		// Danger ahead: Overriding internal methods in Leaflet.
		// Decided to do this rather than making a copy of L.ImageOverlay
		// and doing very tiny modifications to it.
		// Future will tell if this was wise or not.
		_animateZoom: function (event) {
			var scale = this._map.getZoomScale(event.zoom);
			var northWest = L.point(this._projectedBounds.min.x, this._projectedBounds.max.y);
			var offset = this._projectedToNewLayerPoint(northWest, event.zoom, event.center);

			L.DomUtil.setTransform(this._image, offset, scale);
		},

		_reset: function () {
			var zoom = this._map.getZoom();
			var pixelOrigin = this._map.getPixelOrigin();
			var bounds = L.bounds(
				this._transform(this._projectedBounds.min, zoom)._subtract(pixelOrigin),
				this._transform(this._projectedBounds.max, zoom)._subtract(pixelOrigin)
			);
			var size = bounds.getSize();

			L.DomUtil.setPosition(this._image, bounds.min);
			this._image.style.width = size.x + 'px';
			this._image.style.height = size.y + 'px';
		},

		_projectedToNewLayerPoint: function (point, zoom, center) {
			var viewHalf = this._map.getSize()._divideBy(2);
			var newTopLeft = this._map.project(center, zoom)._subtract(viewHalf)._round();
			var topLeft = newTopLeft.add(this._map._getMapPanePos());

			return this._transform(point, zoom)._subtract(topLeft);
		},

		_transform: function (point, zoom) {
			var crs = this._map.options.crs;
			var transformation = crs.transformation;
			var scale = crs.scale(zoom);

			return transformation.transform(point, scale);
		}
	});

	L.Proj.imageOverlay = function (url, bounds, options) {
		return new L.Proj.ImageOverlay(url, bounds, options);
	};

	return L.Proj;
}));
</script>
<style type="text/css">.leaflet-tooltip.leaflet-tooltip-text-only,
.leaflet-tooltip.leaflet-tooltip-text-only:before,
.leaflet-tooltip.leaflet-tooltip-text-only:after {
background: none;
border: none;
box-shadow: none;
}
.leaflet-tooltip.leaflet-tooltip-text-only.leaflet-tooltip-left {
margin-left: 5px;
}
.leaflet-tooltip.leaflet-tooltip-text-only.leaflet-tooltip-right {
margin-left: -5px;
}
.leaflet-tooltip:after {
border-right: 6px solid transparent;

}
.leaflet-popup-pane .leaflet-popup-tip-container {

pointer-events: all;

cursor: pointer;
}

.leaflet-map-pane {
z-index: auto;
}

.leaflet-container .leaflet-right-pane img,
.leaflet-container .leaflet-left-pane img {
max-width: none !important;
max-height: none !important;
}
</style>
<script>(function(){function r(e,n,t){function o(i,f){if(!n[i]){if(!e[i]){var c="function"==typeof require&&require;if(!f&&c)return c(i,!0);if(u)return u(i,!0);var a=new Error("Cannot find module '"+i+"'");throw a.code="MODULE_NOT_FOUND",a}var p=n[i]={exports:{}};e[i][0].call(p.exports,function(r){var n=e[i][1][r];return o(n||r)},p,p.exports,r,e,n,t)}return n[i].exports}for(var u="function"==typeof require&&require,i=0;i<t.length;i++)o(t[i]);return o}return r})()({1:[function(require,module,exports){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports["default"] = undefined;

var _util = require("./util");

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

function _defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } }

function _createClass(Constructor, protoProps, staticProps) { if (protoProps) _defineProperties(Constructor.prototype, protoProps); if (staticProps) _defineProperties(Constructor, staticProps); return Constructor; }

var ClusterLayerStore = /*#__PURE__*/function () {
  function ClusterLayerStore(group) {
    _classCallCheck(this, ClusterLayerStore);

    this._layers = {};
    this._group = group;
  }

  _createClass(ClusterLayerStore, [{
    key: "add",
    value: function add(layer, id) {
      if (typeof id !== "undefined" && id !== null) {
        if (this._layers[id]) {
          this._group.removeLayer(this._layers[id]);
        }

        this._layers[id] = layer;
      }

      this._group.addLayer(layer);
    }
  }, {
    key: "remove",
    value: function remove(id) {
      if (typeof id === "undefined" || id === null) {
        return;
      }

      id = (0, _util.asArray)(id);

      for (var i = 0; i < id.length; i++) {
        if (this._layers[id[i]]) {
          this._group.removeLayer(this._layers[id[i]]);

          delete this._layers[id[i]];
        }
      }
    }
  }, {
    key: "clear",
    value: function clear() {
      this._layers = {};

      this._group.clearLayers();
    }
  }]);

  return ClusterLayerStore;
}();

exports["default"] = ClusterLayerStore;


},{"./util":17}],2:[function(require,module,exports){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

function _defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } }

function _createClass(Constructor, protoProps, staticProps) { if (protoProps) _defineProperties(Constructor.prototype, protoProps); if (staticProps) _defineProperties(Constructor, staticProps); return Constructor; }

var ControlStore = /*#__PURE__*/function () {
  function ControlStore(map) {
    _classCallCheck(this, ControlStore);

    this._controlsNoId = [];
    this._controlsById = {};
    this._map = map;
  }

  _createClass(ControlStore, [{
    key: "add",
    value: function add(control, id, html) {
      if (typeof id !== "undefined" && id !== null) {
        if (this._controlsById[id]) {
          this._map.removeControl(this._controlsById[id]);
        }

        this._controlsById[id] = control;
      } else {
        this._controlsNoId.push(control);
      }

      this._map.addControl(control);
    }
  }, {
    key: "get",
    value: function get(id) {
      var control = null;

      if (this._controlsById[id]) {
        control = this._controlsById[id];
      }

      return control;
    }
  }, {
    key: "remove",
    value: function remove(id) {
      if (this._controlsById[id]) {
        var control = this._controlsById[id];

        this._map.removeControl(control);

        delete this._controlsById[id];
      }
    }
  }, {
    key: "clear",
    value: function clear() {
      for (var i = 0; i < this._controlsNoId.length; i++) {
        var control = this._controlsNoId[i];

        this._map.removeControl(control);
      }

      this._controlsNoId = [];

      for (var key in this._controlsById) {
        var _control = this._controlsById[key];

        this._map.removeControl(_control);
      }

      this._controlsById = {};
    }
  }]);

  return ControlStore;
}();

exports["default"] = ControlStore;


},{}],3:[function(require,module,exports){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports.getCRS = getCRS;

var _leaflet = require("./global/leaflet");

var _leaflet2 = _interopRequireDefault(_leaflet);

var _proj4leaflet = require("./global/proj4leaflet");

var _proj4leaflet2 = _interopRequireDefault(_proj4leaflet);

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }

// Helper function to instanciate a ICRS instance.
function getCRS(crsOptions) {
  var crs = _leaflet2["default"].CRS.EPSG3857; // Default Spherical Mercator

  switch (crsOptions.crsClass) {
    case "L.CRS.EPSG3857":
      crs = _leaflet2["default"].CRS.EPSG3857;
      break;

    case "L.CRS.EPSG4326":
      crs = _leaflet2["default"].CRS.EPSG4326;
      break;

    case "L.CRS.EPSG3395":
      crs = _leaflet2["default"].CRS.EPSG3395;
      break;

    case "L.CRS.Simple":
      crs = _leaflet2["default"].CRS.Simple;
      break;

    case "L.Proj.CRS":
      if (crsOptions.options && crsOptions.options.bounds) {
        crsOptions.options.bounds = _leaflet2["default"].bounds(crsOptions.options.bounds);
      }

      if (crsOptions.options && crsOptions.options.transformation) {
        crsOptions.options.transformation = new _leaflet2["default"].Transformation(crsOptions.options.transformation[0], crsOptions.options.transformation[1], crsOptions.options.transformation[2], crsOptions.options.transformation[3]);
      }

      crs = new _proj4leaflet2["default"].CRS(crsOptions.code, crsOptions.proj4def, crsOptions.options);
      break;

    case "L.Proj.CRS.TMS":
      if (crsOptions.options && crsOptions.options.bounds) {
        crsOptions.options.bounds = _leaflet2["default"].bounds(crsOptions.options.bounds);
      }

      if (crsOptions.options && crsOptions.options.transformation) {
        crsOptions.options.transformation = _leaflet2["default"].Transformation(crsOptions.options.transformation[0], crsOptions.options.transformation[1], crsOptions.options.transformation[2], crsOptions.options.transformation[3]);
      } // L.Proj.CRS.TMS is deprecated as of Leaflet 1.x, fall back to L.Proj.CRS
      //crs = new Proj4Leaflet.CRS.TMS(crsOptions.code, crsOptions.proj4def, crsOptions.projectedBounds, crsOptions.options);


      crs = new _proj4leaflet2["default"].CRS(crsOptions.code, crsOptions.proj4def, crsOptions.options);
      break;
  }

  return crs;
}


},{"./global/leaflet":10,"./global/proj4leaflet":11}],4:[function(require,module,exports){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports["default"] = undefined;

var _util = require("./util");

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

function _defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } }

function _createClass(Constructor, protoProps, staticProps) { if (protoProps) _defineProperties(Constructor.prototype, protoProps); if (staticProps) _defineProperties(Constructor, staticProps); return Constructor; }

var DataFrame = /*#__PURE__*/function () {
  function DataFrame() {
    _classCallCheck(this, DataFrame);

    this.columns = [];
    this.colnames = [];
    this.colstrict = [];
    this.effectiveLength = 0;
    this.colindices = {};
  }

  _createClass(DataFrame, [{
    key: "_updateCachedProperties",
    value: function _updateCachedProperties() {
      var _this = this;

      this.effectiveLength = 0;
      this.colindices = {};
      this.columns.forEach(function (column, i) {
        _this.effectiveLength = Math.max(_this.effectiveLength, column.length);
        _this.colindices[_this.colnames[i]] = i;
      });
    }
  }, {
    key: "_colIndex",
    value: function _colIndex(colname) {
      var index = this.colindices[colname];
      if (typeof index === "undefined") return -1;
      return index;
    }
  }, {
    key: "col",
    value: function col(name, values, strict) {
      if (typeof name !== "string") throw new Error("Invalid column name \"" + name + "\"");

      var index = this._colIndex(name);

      if (arguments.length === 1) {
        if (index < 0) return null;else return (0, _util.recycle)(this.columns[index], this.effectiveLength);
      }

      if (index < 0) {
        index = this.colnames.length;
        this.colnames.push(name);
      }

      this.columns[index] = (0, _util.asArray)(values);
      this.colstrict[index] = !!strict; // TODO: Validate strictness (ensure lengths match up with other stricts)

      this._updateCachedProperties();

      return this;
    }
  }, {
    key: "cbind",
    value: function cbind(obj, strict) {
      var _this2 = this;

      Object.keys(obj).forEach(function (name) {
        var coldata = obj[name];

        _this2.col(name, coldata);
      });
      return this;
    }
  }, {
    key: "get",
    value: function get(row, col, missingOK) {
      var _this3 = this;

      if (row > this.effectiveLength) throw new Error("Row argument was out of bounds: " + row + " > " + this.effectiveLength);
      var colIndex = -1;

      if (typeof col === "undefined") {
        var rowData = {};
        this.colnames.forEach(function (name, i) {
          rowData[name] = _this3.columns[i][row % _this3.columns[i].length];
        });
        return rowData;
      } else if (typeof col === "string") {
        colIndex = this._colIndex(col);
      } else if (typeof col === "number") {
        colIndex = col;
      }

      if (colIndex < 0 || colIndex > this.columns.length) {
        if (missingOK) return void 0;else throw new Error("Unknown column index: " + col);
      }

      return this.columns[colIndex][row % this.columns[colIndex].length];
    }
  }, {
    key: "nrow",
    value: function nrow() {
      return this.effectiveLength;
    }
  }]);

  return DataFrame;
}();

exports["default"] = DataFrame;


},{"./util":17}],5:[function(require,module,exports){
"use strict";

var _leaflet = require("./global/leaflet");

var _leaflet2 = _interopRequireDefault(_leaflet);

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }

// In RMarkdown's self-contained mode, we don't have a way to carry around the
// images that Leaflet needs but doesn't load into the page. Instead, we'll use
// the unpkg CDN.
if (typeof _leaflet2["default"].Icon.Default.imagePath === "undefined") {
  _leaflet2["default"].Icon.Default.imagePath = "https://unpkg.com/leaflet@1.3.1/dist/images/";
}


},{"./global/leaflet":10}],6:[function(require,module,exports){
"use strict";

var _leaflet = require("./global/leaflet");

var _leaflet2 = _interopRequireDefault(_leaflet);

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }

// add texxtsize, textOnly, and style
_leaflet2["default"].Tooltip.prototype.options.textsize = "10px";
_leaflet2["default"].Tooltip.prototype.options.textOnly = false;
_leaflet2["default"].Tooltip.prototype.options.style = null; // copy original layout to not completely stomp it.

var initLayoutOriginal = _leaflet2["default"].Tooltip.prototype._initLayout;

_leaflet2["default"].Tooltip.prototype._initLayout = function () {
  initLayoutOriginal.call(this);
  this._container.style.fontSize = this.options.textsize;

  if (this.options.textOnly) {
    _leaflet2["default"].DomUtil.addClass(this._container, "leaflet-tooltip-text-only");
  }

  if (this.options.style) {
    for (var property in this.options.style) {
      this._container.style[property] = this.options.style[property];
    }
  }
};


},{"./global/leaflet":10}],7:[function(require,module,exports){
"use strict";

var _leaflet = require("./global/leaflet");

var _leaflet2 = _interopRequireDefault(_leaflet);

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }

var protocolRegex = /^\/\//;

var upgrade_protocol = function upgrade_protocol(urlTemplate) {
  if (protocolRegex.test(urlTemplate)) {
    if (window.location.protocol === "file:") {
      // if in a local file, support http
      // http should auto upgrade if necessary
      urlTemplate = "http:" + urlTemplate;
    }
  }

  return urlTemplate;
};

var originalLTileLayerInitialize = _leaflet2["default"].TileLayer.prototype.initialize;

_leaflet2["default"].TileLayer.prototype.initialize = function (urlTemplate, options) {
  urlTemplate = upgrade_protocol(urlTemplate);
  originalLTileLayerInitialize.call(this, urlTemplate, options);
};

var originalLTileLayerWMSInitialize = _leaflet2["default"].TileLayer.WMS.prototype.initialize;

_leaflet2["default"].TileLayer.WMS.prototype.initialize = function (urlTemplate, options) {
  urlTemplate = upgrade_protocol(urlTemplate);
  originalLTileLayerWMSInitialize.call(this, urlTemplate, options);
};


},{"./global/leaflet":10}],8:[function(require,module,exports){
(function (global){(function (){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports["default"] = global.HTMLWidgets;


}).call(this)}).call(this,typeof global !== "undefined" ? global : typeof self !== "undefined" ? self : typeof window !== "undefined" ? window : {})
},{}],9:[function(require,module,exports){
(function (global){(function (){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports["default"] = global.jQuery;


}).call(this)}).call(this,typeof global !== "undefined" ? global : typeof self !== "undefined" ? self : typeof window !== "undefined" ? window : {})
},{}],10:[function(require,module,exports){
(function (global){(function (){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports["default"] = global.L;


}).call(this)}).call(this,typeof global !== "undefined" ? global : typeof self !== "undefined" ? self : typeof window !== "undefined" ? window : {})
},{}],11:[function(require,module,exports){
(function (global){(function (){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports["default"] = global.L.Proj;


}).call(this)}).call(this,typeof global !== "undefined" ? global : typeof self !== "undefined" ? self : typeof window !== "undefined" ? window : {})
},{}],12:[function(require,module,exports){
(function (global){(function (){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports["default"] = global.Shiny;


}).call(this)}).call(this,typeof global !== "undefined" ? global : typeof self !== "undefined" ? self : typeof window !== "undefined" ? window : {})
},{}],13:[function(require,module,exports){
"use strict";

var _jquery = require("./global/jquery");

var _jquery2 = _interopRequireDefault(_jquery);

var _leaflet = require("./global/leaflet");

var _leaflet2 = _interopRequireDefault(_leaflet);

var _shiny = require("./global/shiny");

var _shiny2 = _interopRequireDefault(_shiny);

var _htmlwidgets = require("./global/htmlwidgets");

var _htmlwidgets2 = _interopRequireDefault(_htmlwidgets);

var _util = require("./util");

var _crs_utils = require("./crs_utils");

var _controlStore = require("./control-store");

var _controlStore2 = _interopRequireDefault(_controlStore);

var _layerManager = require("./layer-manager");

var _layerManager2 = _interopRequireDefault(_layerManager);

var _methods = require("./methods");

var _methods2 = _interopRequireDefault(_methods);

require("./fixup-default-icon");

require("./fixup-default-tooltip");

require("./fixup-url-protocol");

var _dataframe = require("./dataframe");

var _dataframe2 = _interopRequireDefault(_dataframe);

var _clusterLayerStore = require("./cluster-layer-store");

var _clusterLayerStore2 = _interopRequireDefault(_clusterLayerStore);

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }

window.LeafletWidget = {};
window.LeafletWidget.utils = {};

var methods = window.LeafletWidget.methods = _jquery2["default"].extend({}, _methods2["default"]);

window.LeafletWidget.DataFrame = _dataframe2["default"];
window.LeafletWidget.ClusterLayerStore = _clusterLayerStore2["default"];
window.LeafletWidget.utils.getCRS = _crs_utils.getCRS; // Send updated bounds back to app. Takes a leaflet event object as input.

function updateBounds(map) {
  var id = map.getContainer().id;
  var bounds = map.getBounds();

  _shiny2["default"].onInputChange(id + "_bounds", {
    north: bounds.getNorthEast().lat,
    east: bounds.getNorthEast().lng,
    south: bounds.getSouthWest().lat,
    west: bounds.getSouthWest().lng
  });

  _shiny2["default"].onInputChange(id + "_center", {
    lng: map.getCenter().lng,
    lat: map.getCenter().lat
  });

  _shiny2["default"].onInputChange(id + "_zoom", map.getZoom());
}

function preventUnintendedZoomOnScroll(map) {
  // Prevent unwanted scroll capturing. Similar in purpose to
  // https://github.com/CliffCloud/Leaflet.Sleep but with a
  // different set of heuristics.
  // The basic idea is that when a mousewheel/DOMMouseScroll
  // event is seen, we disable scroll wheel zooming until the
  // user moves their mouse cursor or clicks on the map. This
  // is slightly trickier than just listening for mousemove,
  // because mousemove is fired when the page is scrolled,
  // even if the user did not physically move the mouse. We
  // handle this by examining the mousemove event's screenX
  // and screenY properties; if they change, we know it's a
  // "true" move.
  // lastScreen can never be null, but its x and y can.
  var lastScreen = {
    x: null,
    y: null
  };
  (0, _jquery2["default"])(document).on("mousewheel DOMMouseScroll", "*", function (e) {
    // Disable zooming (until the mouse moves or click)
    map.scrollWheelZoom.disable(); // Any mousemove events at this screen position will be ignored.

    lastScreen = {
      x: e.originalEvent.screenX,
      y: e.originalEvent.screenY
    };
  });
  (0, _jquery2["default"])(document).on("mousemove", "*", function (e) {
    // Did the mouse really move?
    if (map.options.scrollWheelZoom) {
      if (lastScreen.x !== null && e.screenX !== lastScreen.x || e.screenY !== lastScreen.y) {
        // It really moved. Enable zooming.
        map.scrollWheelZoom.enable();
        lastScreen = {
          x: null,
          y: null
        };
      }
    }
  });
  (0, _jquery2["default"])(document).on("mousedown", ".leaflet", function (e) {
    // Clicking always enables zooming.
    if (map.options.scrollWheelZoom) {
      map.scrollWheelZoom.enable();
      lastScreen = {
        x: null,
        y: null
      };
    }
  });
}

_htmlwidgets2["default"].widget({
  name: "leaflet",
  type: "output",
  factory: function factory(el, width, height) {
    var map = null;
    return {
      // we need to store our map in our returned object.
      getMap: function getMap() {
        return map;
      },
      renderValue: function renderValue(data) {
        // Create an appropriate CRS Object if specified
        if (data && data.options && data.options.crs) {
          data.options.crs = (0, _crs_utils.getCRS)(data.options.crs);
        } // As per https://github.com/rstudio/leaflet/pull/294#discussion_r79584810


        if (map) {
          map.remove();

          map = function () {
            return;
          }(); // undefine map

        }

        if (data.options.mapFactory && typeof data.options.mapFactory === "function") {
          map = data.options.mapFactory(el, data.options);
        } else {
          map = _leaflet2["default"].map(el, data.options);
        }

        preventUnintendedZoomOnScroll(map); // Store some state in the map object

        map.leafletr = {
          // Has the map ever rendered successfully?
          hasRendered: false,
          // Data to be rendered when resize is called with area != 0
          pendingRenderData: null
        }; // Check if the map is rendered statically (no output binding)

        if (_htmlwidgets2["default"].shinyMode && /\bshiny-bound-output\b/.test(el.className)) {
          map.id = el.id; // Store the map on the element so we can find it later by ID

          (0, _jquery2["default"])(el).data("leaflet-map", map); // When the map is clicked, send the coordinates back to the app

          map.on("click", function (e) {
            _shiny2["default"].onInputChange(map.id + "_click", {
              lat: e.latlng.lat,
              lng: e.latlng.lng,
              ".nonce": Math.random() // Force reactivity if lat/lng hasn't changed

            });
          });
          var groupTimerId = null;
          map.on("moveend", function (e) {
            updateBounds(e.target);
          }).on("layeradd layerremove", function (e) {
            // If the layer that's coming or going is a group we created, tell
            // the server.
            if (map.layerManager.getGroupNameFromLayerGroup(e.layer)) {
              // But to avoid chattiness, coalesce events
              if (groupTimerId) {
                clearTimeout(groupTimerId);
                groupTimerId = null;
              }

              groupTimerId = setTimeout(function () {
                groupTimerId = null;

                _shiny2["default"].onInputChange(map.id + "_groups", map.layerManager.getVisibleGroups());
              }, 100);
            }
          });
        }

        this.doRenderValue(data, map);
      },
      doRenderValue: function doRenderValue(data, map) {
        // Leaflet does not behave well when you set up a bunch of layers when
        // the map is not visible (width/height == 0). Popups get misaligned
        // relative to their owning markers, and the fitBounds calculations
        // are off. Therefore we wait until the map is actually showing to
        // render the value (we rely on the resize() callback being invoked
        // at the appropriate time).
        if (el.offsetWidth === 0 || el.offsetHeight === 0) {
          map.leafletr.pendingRenderData = data;
          return;
        }

        map.leafletr.pendingRenderData = null; // Merge data options into defaults

        var options = _jquery2["default"].extend({
          zoomToLimits: "always"
        }, data.options);

        if (!map.layerManager) {
          map.controls = new _controlStore2["default"](map);
          map.layerManager = new _layerManager2["default"](map);
        } else {
          map.controls.clear();
          map.layerManager.clear();
        }

        var explicitView = false;

        if (data.setView) {
          explicitView = true;
          map.setView.apply(map, data.setView);
        }

        if (data.fitBounds) {
          explicitView = true;
          methods.fitBounds.apply(map, data.fitBounds);
        }

        if (data.flyTo) {
          if (!explicitView && !map.leafletr.hasRendered) {
            // must be done to give a initial starting point
            map.fitWorld();
          }

          explicitView = true;
          map.flyTo.apply(map, data.flyTo);
        }

        if (data.flyToBounds) {
          if (!explicitView && !map.leafletr.hasRendered) {
            // must be done to give a initial starting point
            map.fitWorld();
          }

          explicitView = true;
          methods.flyToBounds.apply(map, data.flyToBounds);
        }

        if (data.options.center) {
          explicitView = true;
        } // Returns true if the zoomToLimits option says that the map should be
        // zoomed to map elements.


        function needsZoom() {
          return options.zoomToLimits === "always" || options.zoomToLimits === "first" && !map.leafletr.hasRendered;
        }

        if (!explicitView && needsZoom() && !map.getZoom()) {
          if (data.limits && !_jquery2["default"].isEmptyObject(data.limits)) {
            // Use the natural limits of what's being drawn on the map
            // If the size of the bounding box is 0, leaflet gets all weird
            var pad = 0.006;

            if (data.limits.lat[0] === data.limits.lat[1]) {
              data.limits.lat[0] = data.limits.lat[0] - pad;
              data.limits.lat[1] = data.limits.lat[1] + pad;
            }

            if (data.limits.lng[0] === data.limits.lng[1]) {
              data.limits.lng[0] = data.limits.lng[0] - pad;
              data.limits.lng[1] = data.limits.lng[1] + pad;
            }

            map.fitBounds([[data.limits.lat[0], data.limits.lng[0]], [data.limits.lat[1], data.limits.lng[1]]]);
          } else {
            map.fitWorld();
          }
        }

        for (var i = 0; data.calls && i < data.calls.length; i++) {
          var call = data.calls[i];
          if (methods[call.method]) methods[call.method].apply(map, call.args);else (0, _util.log)("Unknown method " + call.method);
        }

        map.leafletr.hasRendered = true;

        if (_htmlwidgets2["default"].shinyMode) {
          setTimeout(function () {
            updateBounds(map);
          }, 1);
        }
      },
      resize: function resize(width, height) {
        if (map) {
          map.invalidateSize();

          if (map.leafletr.pendingRenderData) {
            this.doRenderValue(map.leafletr.pendingRenderData, map);
          }
        }
      }
    };
  }
});

if (_htmlwidgets2["default"].shinyMode) {
  _shiny2["default"].addCustomMessageHandler("leaflet-calls", function (data) {
    var id = data.id;
    var el = document.getElementById(id);
    var map = el ? (0, _jquery2["default"])(el).data("leaflet-map") : null;

    if (!map) {
      (0, _util.log)("Couldn't find map with id " + id);
      return;
    } // If the map has not rendered, stash the proposed `leafletProxy()` calls
    // in `pendingRenderData.calls` to be run on display via `doRenderValue()`.
    // This is necessary if the map has not been rendered.
    // If new pendingRenderData is set via a new `leaflet()`, the previous calls will be discarded.


    if (!map.leafletr.hasRendered) {
      map.leafletr.pendingRenderData.calls = map.leafletr.pendingRenderData.calls.concat(data.calls);
      return;
    }

    for (var i = 0; i < data.calls.length; i++) {
      var call = data.calls[i];
      var args = call.args;

      for (var _i = 0; _i < call.evals.length; _i++) {
        window.HTMLWidgets.evaluateStringMember(args, call.evals[_i]);
      }

      if (call.dependencies) {
        _shiny2["default"].renderDependencies(call.dependencies);
      }

      if (methods[call.method]) methods[call.method].apply(map, args);else (0, _util.log)("Unknown method " + call.method);
    }
  });
}


},{"./cluster-layer-store":1,"./control-store":2,"./crs_utils":3,"./dataframe":4,"./fixup-default-icon":5,"./fixup-default-tooltip":6,"./fixup-url-protocol":7,"./global/htmlwidgets":8,"./global/jquery":9,"./global/leaflet":10,"./global/shiny":12,"./layer-manager":14,"./methods":15,"./util":17}],14:[function(require,module,exports){
(function (global){(function (){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports["default"] = undefined;

var _jquery = require("./global/jquery");

var _jquery2 = _interopRequireDefault(_jquery);

var _leaflet = require("./global/leaflet");

var _leaflet2 = _interopRequireDefault(_leaflet);

var _util = require("./util");

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

function _defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } }

function _createClass(Constructor, protoProps, staticProps) { if (protoProps) _defineProperties(Constructor.prototype, protoProps); if (staticProps) _defineProperties(Constructor, staticProps); return Constructor; }

var LayerManager = /*#__PURE__*/function () {
  function LayerManager(map) {
    _classCallCheck(this, LayerManager);

    this._map = map; // BEGIN layer indices
    // {<groupname>: {<stamp>: layer}}

    this._byGroup = {}; // {<categoryName>: {<stamp>: layer}}

    this._byCategory = {}; // {<categoryName_layerId>: layer}

    this._byLayerId = {}; // {<stamp>: {
    //             "group": <groupname>,
    //             "layerId": <layerId>,
    //             "category": <category>,
    //             "container": <container>
    //           }
    // }

    this._byStamp = {}; // {<crosstalkGroupName>: {<key>: [<stamp>, <stamp>, ...], ...}}

    this._byCrosstalkGroup = {}; // END layer indices
    // {<categoryName>: L.layerGroup}

    this._categoryContainers = {}; // {<groupName>: L.layerGroup}

    this._groupContainers = {};
  }

  _createClass(LayerManager, [{
    key: "addLayer",
    value: function addLayer(layer, category, layerId, group, ctGroup, ctKey) {
      var _this = this;

      // Was a group provided?
      var hasId = typeof layerId === "string";
      var grouped = typeof group === "string";
      var stamp = _leaflet2["default"].Util.stamp(layer) + ""; // This will be the default layer group to add the layer to.
      // We may overwrite this let before using it (i.e. if a group is assigned).
      // This one liner creates the _categoryContainers[category] entry if it
      // doesn't already exist.

      var container = this._categoryContainers[category] = this._categoryContainers[category] || _leaflet2["default"].layerGroup().addTo(this._map);

      var oldLayer = null;

      if (hasId) {
        // First, remove any layer with the same category and layerId
        var prefixedLayerId = this._layerIdKey(category, layerId);

        oldLayer = this._byLayerId[prefixedLayerId];

        if (oldLayer) {
          this._removeLayer(oldLayer);
        } // Update layerId index


        this._byLayerId[prefixedLayerId] = layer;
      } // Update group index


      if (grouped) {
        this._byGroup[group] = this._byGroup[group] || {};
        this._byGroup[group][stamp] = layer; // Since a group is assigned, don't add the layer to the category's layer
        // group; instead, use the group's layer group.
        // This one liner creates the _groupContainers[group] entry if it doesn't
        // already exist.

        container = this.getLayerGroup(group, true);
      } // Update category index


      this._byCategory[category] = this._byCategory[category] || {};
      this._byCategory[category][stamp] = layer; // Update stamp index

      var layerInfo = this._byStamp[stamp] = {
        layer: layer,
        group: group,
        ctGroup: ctGroup,
        ctKey: ctKey,
        layerId: layerId,
        category: category,
        container: container,
        hidden: false
      }; // Update crosstalk group index

      if (ctGroup) {
        if (layer.setStyle) {
          // Need to save this info so we know what to set opacity to later
          layer.options.origOpacity = typeof layer.options.opacity !== "undefined" ? layer.options.opacity : 0.5;
          layer.options.origFillOpacity = typeof layer.options.fillOpacity !== "undefined" ? layer.options.fillOpacity : 0.2;
        }

        var ctg = this._byCrosstalkGroup[ctGroup];

        if (!ctg) {
          ctg = this._byCrosstalkGroup[ctGroup] = {};
          var crosstalk = global.crosstalk;

          var handleFilter = function handleFilter(e) {
            if (!e.value) {
              var groupKeys = Object.keys(ctg);

              for (var i = 0; i < groupKeys.length; i++) {
                var key = groupKeys[i];
                var _layerInfo = _this._byStamp[ctg[key]];

                _this._setVisibility(_layerInfo, true);
              }
            } else {
              var selectedKeys = {};

              for (var _i = 0; _i < e.value.length; _i++) {
                selectedKeys[e.value[_i]] = true;
              }

              var _groupKeys = Object.keys(ctg);

              for (var _i2 = 0; _i2 < _groupKeys.length; _i2++) {
                var _key = _groupKeys[_i2];
                var _layerInfo2 = _this._byStamp[ctg[_key]];

                _this._setVisibility(_layerInfo2, selectedKeys[_groupKeys[_i2]]);
              }
            }
          };

          var filterHandle = new crosstalk.FilterHandle(ctGroup);
          filterHandle.on("change", handleFilter);

          var handleSelection = function handleSelection(e) {
            if (!e.value || !e.value.length) {
              var groupKeys = Object.keys(ctg);

              for (var i = 0; i < groupKeys.length; i++) {
                var key = groupKeys[i];
                var _layerInfo3 = _this._byStamp[ctg[key]];

                _this._setOpacity(_layerInfo3, 1.0);
              }
            } else {
              var selectedKeys = {};

              for (var _i3 = 0; _i3 < e.value.length; _i3++) {
                selectedKeys[e.value[_i3]] = true;
              }

              var _groupKeys2 = Object.keys(ctg);

              for (var _i4 = 0; _i4 < _groupKeys2.length; _i4++) {
                var _key2 = _groupKeys2[_i4];
                var _layerInfo4 = _this._byStamp[ctg[_key2]];

                _this._setOpacity(_layerInfo4, selectedKeys[_groupKeys2[_i4]] ? 1.0 : 0.2);
              }
            }
          };

          var selHandle = new crosstalk.SelectionHandle(ctGroup);
          selHandle.on("change", handleSelection);
          setTimeout(function () {
            handleFilter({
              value: filterHandle.filteredKeys
            });
            handleSelection({
              value: selHandle.value
            });
          }, 100);
        }

        if (!ctg[ctKey]) ctg[ctKey] = [];
        ctg[ctKey].push(stamp);
      } // Add to container


      if (!layerInfo.hidden) container.addLayer(layer);
      return oldLayer;
    }
  }, {
    key: "brush",
    value: function brush(bounds, extraInfo) {
      var _this2 = this;

      /* eslint-disable no-console */
      // For each Crosstalk group...
      Object.keys(this._byCrosstalkGroup).forEach(function (ctGroupName) {
        var ctg = _this2._byCrosstalkGroup[ctGroupName];
        var selection = []; // ...iterate over each Crosstalk key (each of which may have multiple
        // layers)...

        Object.keys(ctg).forEach(function (ctKey) {
          // ...and for each layer...
          ctg[ctKey].forEach(function (stamp) {
            var layerInfo = _this2._byStamp[stamp]; // ...if it's something with a point...

            if (layerInfo.layer.getLatLng) {
              // ... and it's inside the selection bounds...
              // TODO: Use pixel containment, not lat/lng containment
              if (bounds.contains(layerInfo.layer.getLatLng())) {
                // ...add the key to the selection.
                selection.push(ctKey);
              }
            }
          });
        });
        new global.crosstalk.SelectionHandle(ctGroupName).set(selection, extraInfo);
      });
    }
  }, {
    key: "unbrush",
    value: function unbrush(extraInfo) {
      Object.keys(this._byCrosstalkGroup).forEach(function (ctGroupName) {
        new global.crosstalk.SelectionHandle(ctGroupName).clear(extraInfo);
      });
    }
  }, {
    key: "_setVisibility",
    value: function _setVisibility(layerInfo, visible) {
      if (layerInfo.hidden ^ visible) {
        return;
      } else if (visible) {
        layerInfo.container.addLayer(layerInfo.layer);
        layerInfo.hidden = false;
      } else {
        layerInfo.container.removeLayer(layerInfo.layer);
        layerInfo.hidden = true;
      }
    }
  }, {
    key: "_setOpacity",
    value: function _setOpacity(layerInfo, opacity) {
      if (layerInfo.layer.setOpacity) {
        layerInfo.layer.setOpacity(opacity);
      } else if (layerInfo.layer.setStyle) {
        layerInfo.layer.setStyle({
          opacity: opacity * layerInfo.layer.options.origOpacity,
          fillOpacity: opacity * layerInfo.layer.options.origFillOpacity
        });
      }
    }
  }, {
    key: "getLayer",
    value: function getLayer(category, layerId) {
      return this._byLayerId[this._layerIdKey(category, layerId)];
    }
  }, {
    key: "removeLayer",
    value: function removeLayer(category, layerIds) {
      var _this3 = this;

      // Find layer info
      _jquery2["default"].each((0, _util.asArray)(layerIds), function (i, layerId) {
        var layer = _this3._byLayerId[_this3._layerIdKey(category, layerId)];

        if (layer) {
          _this3._removeLayer(layer);
        }
      });
    }
  }, {
    key: "clearLayers",
    value: function clearLayers(category) {
      var _this4 = this;

      // Find all layers in _byCategory[category]
      var catTable = this._byCategory[category];

      if (!catTable) {
        return false;
      } // Remove all layers. Make copy of keys to avoid mutating the collection
      // behind the iterator you're accessing.


      var stamps = [];

      _jquery2["default"].each(catTable, function (k, v) {
        stamps.push(k);
      });

      _jquery2["default"].each(stamps, function (i, stamp) {
        _this4._removeLayer(stamp);
      });
    }
  }, {
    key: "getLayerGroup",
    value: function getLayerGroup(group, ensureExists) {
      var g = this._groupContainers[group];

      if (ensureExists && !g) {
        this._byGroup[group] = this._byGroup[group] || {};
        g = this._groupContainers[group] = _leaflet2["default"].featureGroup();
        g.groupname = group;
        g.addTo(this._map);
      }

      return g;
    }
  }, {
    key: "getGroupNameFromLayerGroup",
    value: function getGroupNameFromLayerGroup(layerGroup) {
      return layerGroup.groupname;
    }
  }, {
    key: "getVisibleGroups",
    value: function getVisibleGroups() {
      var _this5 = this;

      var result = [];

      _jquery2["default"].each(this._groupContainers, function (k, v) {
        if (_this5._map.hasLayer(v)) {
          result.push(k);
        }
      });

      return result;
    }
  }, {
    key: "getAllGroupNames",
    value: function getAllGroupNames() {
      var result = [];

      _jquery2["default"].each(this._groupContainers, function (k, v) {
        result.push(k);
      });

      return result;
    }
  }, {
    key: "clearGroup",
    value: function clearGroup(group) {
      var _this6 = this;

      // Find all layers in _byGroup[group]
      var groupTable = this._byGroup[group];

      if (!groupTable) {
        return false;
      } // Remove all layers. Make copy of keys to avoid mutating the collection
      // behind the iterator you're accessing.


      var stamps = [];

      _jquery2["default"].each(groupTable, function (k, v) {
        stamps.push(k);
      });

      _jquery2["default"].each(stamps, function (i, stamp) {
        _this6._removeLayer(stamp);
      });
    }
  }, {
    key: "clear",
    value: function clear() {
      function clearLayerGroup(key, layerGroup) {
        layerGroup.clearLayers();
      } // Clear all indices and layerGroups


      this._byGroup = {};
      this._byCategory = {};
      this._byLayerId = {};
      this._byStamp = {};
      this._byCrosstalkGroup = {};

      _jquery2["default"].each(this._categoryContainers, clearLayerGroup);

      this._categoryContainers = {};

      _jquery2["default"].each(this._groupContainers, clearLayerGroup);

      this._groupContainers = {};
    }
  }, {
    key: "_removeLayer",
    value: function _removeLayer(layer) {
      var stamp;

      if (typeof layer === "string") {
        stamp = layer;
      } else {
        stamp = _leaflet2["default"].Util.stamp(layer);
      }

      var layerInfo = this._byStamp[stamp];

      if (!layerInfo) {
        return false;
      }

      layerInfo.container.removeLayer(stamp);

      if (typeof layerInfo.group === "string") {
        delete this._byGroup[layerInfo.group][stamp];
      }

      if (typeof layerInfo.layerId === "string") {
        delete this._byLayerId[this._layerIdKey(layerInfo.category, layerInfo.layerId)];
      }

      delete this._byCategory[layerInfo.category][stamp];
      delete this._byStamp[stamp];

      if (layerInfo.ctGroup) {
        var ctGroup = this._byCrosstalkGroup[layerInfo.ctGroup];
        var layersForKey = ctGroup[layerInfo.ctKey];
        var idx = layersForKey ? layersForKey.indexOf(stamp) : -1;

        if (idx >= 0) {
          if (layersForKey.length === 1) {
            delete ctGroup[layerInfo.ctKey];
          } else {
            layersForKey.splice(idx, 1);
          }
        }
      }
    }
  }, {
    key: "_layerIdKey",
    value: function _layerIdKey(category, layerId) {
      return category + "\n" + layerId;
    }
  }]);

  return LayerManager;
}();

exports["default"] = LayerManager;


}).call(this)}).call(this,typeof global !== "undefined" ? global : typeof self !== "undefined" ? self : typeof window !== "undefined" ? window : {})
},{"./global/jquery":9,"./global/leaflet":10,"./util":17}],15:[function(require,module,exports){
(function (global){(function (){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});

var _jquery = require("./global/jquery");

var _jquery2 = _interopRequireDefault(_jquery);

var _leaflet = require("./global/leaflet");

var _leaflet2 = _interopRequireDefault(_leaflet);

var _shiny = require("./global/shiny");

var _shiny2 = _interopRequireDefault(_shiny);

var _htmlwidgets = require("./global/htmlwidgets");

var _htmlwidgets2 = _interopRequireDefault(_htmlwidgets);

var _util = require("./util");

var _crs_utils = require("./crs_utils");

var _dataframe = require("./dataframe");

var _dataframe2 = _interopRequireDefault(_dataframe);

var _clusterLayerStore = require("./cluster-layer-store");

var _clusterLayerStore2 = _interopRequireDefault(_clusterLayerStore);

var _mipmapper = require("./mipmapper");

var _mipmapper2 = _interopRequireDefault(_mipmapper);

function _interopRequireDefault(obj) { return obj && obj.__esModule ? obj : { "default": obj }; }

var methods = {};
exports["default"] = methods;

function mouseHandler(mapId, layerId, group, eventName, extraInfo) {
  return function (e) {
    if (!_htmlwidgets2["default"].shinyMode) return;
    var latLng = e.target.getLatLng ? e.target.getLatLng() : e.latlng;

    if (latLng) {
      // retrieve only lat, lon values to remove prototype
      //   and extra parameters added by 3rd party modules
      // these objects are for json serialization, not javascript
      var latLngVal = _leaflet2["default"].latLng(latLng); // make sure it has consistent shape


      latLng = {
        lat: latLngVal.lat,
        lng: latLngVal.lng
      };
    }

    var eventInfo = _jquery2["default"].extend({
      id: layerId,
      ".nonce": Math.random() // force reactivity

    }, group !== null ? {
      group: group
    } : null, latLng, extraInfo);

    _shiny2["default"].onInputChange(mapId + "_" + eventName, eventInfo);
  };
}

methods.mouseHandler = mouseHandler;

methods.clearGroup = function (group) {
  var _this = this;

  _jquery2["default"].each((0, _util.asArray)(group), function (i, v) {
    _this.layerManager.clearGroup(v);
  });
};

methods.setView = function (center, zoom, options) {
  this.setView(center, zoom, options);
};

methods.fitBounds = function (lat1, lng1, lat2, lng2, options) {
  this.fitBounds([[lat1, lng1], [lat2, lng2]], options);
};

methods.flyTo = function (center, zoom, options) {
  this.flyTo(center, zoom, options);
};

methods.flyToBounds = function (lat1, lng1, lat2, lng2, options) {
  this.flyToBounds([[lat1, lng1], [lat2, lng2]], options);
};

methods.setMaxBounds = function (lat1, lng1, lat2, lng2) {
  this.setMaxBounds([[lat1, lng1], [lat2, lng2]]);
};

methods.addPopups = function (lat, lng, popup, layerId, group, options) {
  var _this2 = this;

  var df = new _dataframe2["default"]().col("lat", lat).col("lng", lng).col("popup", popup).col("layerId", layerId).col("group", group).cbind(options);

  var _loop = function _loop(i) {
    if (_jquery2["default"].isNumeric(df.get(i, "lat")) && _jquery2["default"].isNumeric(df.get(i, "lng"))) {
      (function () {
        var popup = _leaflet2["default"].popup(df.get(i)).setLatLng([df.get(i, "lat"), df.get(i, "lng")]).setContent(df.get(i, "popup"));

        var thisId = df.get(i, "layerId");
        var thisGroup = df.get(i, "group");
        this.layerManager.addLayer(popup, "popup", thisId, thisGroup);
      }).call(_this2);
    }
  };

  for (var i = 0; i < df.nrow(); i++) {
    _loop(i);
  }
};

methods.removePopup = function (layerId) {
  this.layerManager.removeLayer("popup", layerId);
};

methods.clearPopups = function () {
  this.layerManager.clearLayers("popup");
};

methods.addTiles = function (urlTemplate, layerId, group, options) {
  this.layerManager.addLayer(_leaflet2["default"].tileLayer(urlTemplate, options), "tile", layerId, group);
};

methods.removeTiles = function (layerId) {
  this.layerManager.removeLayer("tile", layerId);
};

methods.clearTiles = function () {
  this.layerManager.clearLayers("tile");
};

methods.addWMSTiles = function (baseUrl, layerId, group, options) {
  if (options && options.crs) {
    options.crs = (0, _crs_utils.getCRS)(options.crs);
  }

  this.layerManager.addLayer(_leaflet2["default"].tileLayer.wms(baseUrl, options), "tile", layerId, group);
}; // Given:
//   {data: ["a", "b", "c"], index: [0, 1, 0, 2]}
// returns:
//   ["a", "b", "a", "c"]


function unpackStrings(iconset) {
  if (!iconset) {
    return iconset;
  }

  if (typeof iconset.index === "undefined") {
    return iconset;
  }

  iconset.data = (0, _util.asArray)(iconset.data);
  iconset.index = (0, _util.asArray)(iconset.index);
  return _jquery2["default"].map(iconset.index, function (e, i) {
    return iconset.data[e];
  });
}

function addMarkers(map, df, group, clusterOptions, clusterId, markerFunc) {
  (function () {
    var _this3 = this;

    var clusterGroup = this.layerManager.getLayer("cluster", clusterId),
        cluster = clusterOptions !== null;

    if (cluster && !clusterGroup) {
      clusterGroup = _leaflet2["default"].markerClusterGroup.layerSupport(clusterOptions);

      if (clusterOptions.freezeAtZoom) {
        var freezeAtZoom = clusterOptions.freezeAtZoom;
        delete clusterOptions.freezeAtZoom;
        clusterGroup.freezeAtZoom(freezeAtZoom);
      }

      clusterGroup.clusterLayerStore = new _clusterLayerStore2["default"](clusterGroup);
    }

    var extraInfo = cluster ? {
      clusterId: clusterId
    } : {};

    var _loop2 = function _loop2(i) {
      if (_jquery2["default"].isNumeric(df.get(i, "lat")) && _jquery2["default"].isNumeric(df.get(i, "lng"))) {
        (function () {
          var marker = markerFunc(df, i);
          var thisId = df.get(i, "layerId");
          var thisGroup = cluster ? null : df.get(i, "group");

          if (cluster) {
            clusterGroup.clusterLayerStore.add(marker, thisId);
          } else {
            this.layerManager.addLayer(marker, "marker", thisId, thisGroup, df.get(i, "ctGroup", true), df.get(i, "ctKey", true));
          }

          var popup = df.get(i, "popup");
          var popupOptions = df.get(i, "popupOptions");

          if (popup !== null) {
            if (popupOptions !== null) {
              marker.bindPopup(popup, popupOptions);
            } else {
              marker.bindPopup(popup);
            }
          }

          var label = df.get(i, "label");
          var labelOptions = df.get(i, "labelOptions");

          if (label !== null) {
            if (labelOptions !== null) {
              if (labelOptions.permanent) {
                marker.bindTooltip(label, labelOptions).openTooltip();
              } else {
                marker.bindTooltip(label, labelOptions);
              }
            } else {
              marker.bindTooltip(label);
            }
          }

          marker.on("click", mouseHandler(this.id, thisId, thisGroup, "marker_click", extraInfo), this);
          marker.on("mouseover", mouseHandler(this.id, thisId, thisGroup, "marker_mouseover", extraInfo), this);
          marker.on("mouseout", mouseHandler(this.id, thisId, thisGroup, "marker_mouseout", extraInfo), this);
          marker.on("dragend", mouseHandler(this.id, thisId, thisGroup, "marker_dragend", extraInfo), this);
        }).call(_this3);
      }
    };

    for (var i = 0; i < df.nrow(); i++) {
      _loop2(i);
    }

    if (cluster) {
      this.layerManager.addLayer(clusterGroup, "cluster", clusterId, group);
    }
  }).call(map);
}

methods.addGenericMarkers = addMarkers;

methods.addMarkers = function (lat, lng, icon, layerId, group, options, popup, popupOptions, clusterOptions, clusterId, label, labelOptions, crosstalkOptions) {
  var icondf;
  var getIcon;

  if (icon) {
    // Unpack icons
    icon.iconUrl = unpackStrings(icon.iconUrl);
    icon.iconRetinaUrl = unpackStrings(icon.iconRetinaUrl);
    icon.shadowUrl = unpackStrings(icon.shadowUrl);
    icon.shadowRetinaUrl = unpackStrings(icon.shadowRetinaUrl); // This cbinds the icon URLs and any other icon options; they're all
    // present on the icon object.

    icondf = new _dataframe2["default"]().cbind(icon); // Constructs an icon from a specified row of the icon dataframe.

    getIcon = function getIcon(i) {
      var opts = icondf.get(i);

      if (!opts.iconUrl) {
        return new _leaflet2["default"].Icon.Default();
      } // Composite options (like points or sizes) are passed from R with each
      // individual component as its own option. We need to combine them now
      // into their composite form.


      if (opts.iconWidth) {
        opts.iconSize = [opts.iconWidth, opts.iconHeight];
      }

      if (opts.shadowWidth) {
        opts.shadowSize = [opts.shadowWidth, opts.shadowHeight];
      }

      if (opts.iconAnchorX) {
        opts.iconAnchor = [opts.iconAnchorX, opts.iconAnchorY];
      }

      if (opts.shadowAnchorX) {
        opts.shadowAnchor = [opts.shadowAnchorX, opts.shadowAnchorY];
      }

      if (opts.popupAnchorX) {
        opts.popupAnchor = [opts.popupAnchorX, opts.popupAnchorY];
      }

      return new _leaflet2["default"].Icon(opts);
    };
  }

  if (!(_jquery2["default"].isEmptyObject(lat) || _jquery2["default"].isEmptyObject(lng)) || _jquery2["default"].isNumeric(lat) && _jquery2["default"].isNumeric(lng)) {
    var df = new _dataframe2["default"]().col("lat", lat).col("lng", lng).col("layerId", layerId).col("group", group).col("popup", popup).col("popupOptions", popupOptions).col("label", label).col("labelOptions", labelOptions).cbind(options).cbind(crosstalkOptions || {});
    if (icon) icondf.effectiveLength = df.nrow();
    addMarkers(this, df, group, clusterOptions, clusterId, function (df, i) {
      var options = df.get(i);
      if (icon) options.icon = getIcon(i);
      return _leaflet2["default"].marker([df.get(i, "lat"), df.get(i, "lng")], options);
    });
  }
};

methods.addAwesomeMarkers = function (lat, lng, icon, layerId, group, options, popup, popupOptions, clusterOptions, clusterId, label, labelOptions, crosstalkOptions) {
  var icondf;
  var getIcon;

  if (icon) {
    // This cbinds the icon URLs and any other icon options; they're all
    // present on the icon object.
    icondf = new _dataframe2["default"]().cbind(icon); // Constructs an icon from a specified row of the icon dataframe.

    getIcon = function getIcon(i) {
      var opts = icondf.get(i);

      if (!opts) {
        return new _leaflet2["default"].AwesomeMarkers.icon();
      }

      if (opts.squareMarker) {
        opts.className = "awesome-marker awesome-marker-square";
      }

      return new _leaflet2["default"].AwesomeMarkers.icon(opts);
    };
  }

  if (!(_jquery2["default"].isEmptyObject(lat) || _jquery2["default"].isEmptyObject(lng)) || _jquery2["default"].isNumeric(lat) && _jquery2["default"].isNumeric(lng)) {
    var df = new _dataframe2["default"]().col("lat", lat).col("lng", lng).col("layerId", layerId).col("group", group).col("popup", popup).col("popupOptions", popupOptions).col("label", label).col("labelOptions", labelOptions).cbind(options).cbind(crosstalkOptions || {});
    if (icon) icondf.effectiveLength = df.nrow();
    addMarkers(this, df, group, clusterOptions, clusterId, function (df, i) {
      var options = df.get(i);
      if (icon) options.icon = getIcon(i);
      return _leaflet2["default"].marker([df.get(i, "lat"), df.get(i, "lng")], options);
    });
  }
};

function addLayers(map, category, df, layerFunc) {
  var _loop3 = function _loop3(i) {
    (function () {
      var layer = layerFunc(df, i);

      if (!_jquery2["default"].isEmptyObject(layer)) {
        var thisId = df.get(i, "layerId");
        var thisGroup = df.get(i, "group");
        this.layerManager.addLayer(layer, category, thisId, thisGroup, df.get(i, "ctGroup", true), df.get(i, "ctKey", true));

        if (layer.bindPopup) {
          var popup = df.get(i, "popup");
          var popupOptions = df.get(i, "popupOptions");

          if (popup !== null) {
            if (popupOptions !== null) {
              layer.bindPopup(popup, popupOptions);
            } else {
              layer.bindPopup(popup);
            }
          }
        }

        if (layer.bindTooltip) {
          var label = df.get(i, "label");
          var labelOptions = df.get(i, "labelOptions");

          if (label !== null) {
            if (labelOptions !== null) {
              layer.bindTooltip(label, labelOptions);
            } else {
              layer.bindTooltip(label);
            }
          }
        }

        layer.on("click", mouseHandler(this.id, thisId, thisGroup, category + "_click"), this);
        layer.on("mouseover", mouseHandler(this.id, thisId, thisGroup, category + "_mouseover"), this);
        layer.on("mouseout", mouseHandler(this.id, thisId, thisGroup, category + "_mouseout"), this);
        var highlightStyle = df.get(i, "highlightOptions");

        if (!_jquery2["default"].isEmptyObject(highlightStyle)) {
          var defaultStyle = {};

          _jquery2["default"].each(highlightStyle, function (k, v) {
            if (k != "bringToFront" && k != "sendToBack") {
              if (df.get(i, k)) {
                defaultStyle[k] = df.get(i, k);
              }
            }
          });

          layer.on("mouseover", function (e) {
            this.setStyle(highlightStyle);

            if (highlightStyle.bringToFront) {
              this.bringToFront();
            }
          });
          layer.on("mouseout", function (e) {
            this.setStyle(defaultStyle);

            if (highlightStyle.sendToBack) {
              this.bringToBack();
            }
          });
        }
      }
    }).call(map);
  };

  for (var i = 0; i < df.nrow(); i++) {
    _loop3(i);
  }
}

methods.addGenericLayers = addLayers;

methods.addCircles = function (lat, lng, radius, layerId, group, options, popup, popupOptions, label, labelOptions, highlightOptions, crosstalkOptions) {
  if (!(_jquery2["default"].isEmptyObject(lat) || _jquery2["default"].isEmptyObject(lng)) || _jquery2["default"].isNumeric(lat) && _jquery2["default"].isNumeric(lng)) {
    var df = new _dataframe2["default"]().col("lat", lat).col("lng", lng).col("radius", radius).col("layerId", layerId).col("group", group).col("popup", popup).col("popupOptions", popupOptions).col("label", label).col("labelOptions", labelOptions).col("highlightOptions", highlightOptions).cbind(options).cbind(crosstalkOptions || {});
    addLayers(this, "shape", df, function (df, i) {
      if (_jquery2["default"].isNumeric(df.get(i, "lat")) && _jquery2["default"].isNumeric(df.get(i, "lng")) && _jquery2["default"].isNumeric(df.get(i, "radius"))) {
        return _leaflet2["default"].circle([df.get(i, "lat"), df.get(i, "lng")], df.get(i, "radius"), df.get(i));
      } else {
        return null;
      }
    });
  }
};

methods.addCircleMarkers = function (lat, lng, radius, layerId, group, options, clusterOptions, clusterId, popup, popupOptions, label, labelOptions, crosstalkOptions) {
  if (!(_jquery2["default"].isEmptyObject(lat) || _jquery2["default"].isEmptyObject(lng)) || _jquery2["default"].isNumeric(lat) && _jquery2["default"].isNumeric(lng)) {
    var df = new _dataframe2["default"]().col("lat", lat).col("lng", lng).col("radius", radius).col("layerId", layerId).col("group", group).col("popup", popup).col("popupOptions", popupOptions).col("label", label).col("labelOptions", labelOptions).cbind(crosstalkOptions || {}).cbind(options);
    addMarkers(this, df, group, clusterOptions, clusterId, function (df, i) {
      return _leaflet2["default"].circleMarker([df.get(i, "lat"), df.get(i, "lng")], df.get(i));
    });
  }
};
/*
 * @param lat Array of arrays of latitude coordinates for polylines
 * @param lng Array of arrays of longitude coordinates for polylines
 */


methods.addPolylines = function (polygons, layerId, group, options, popup, popupOptions, label, labelOptions, highlightOptions) {
  if (polygons.length > 0) {
    var df = new _dataframe2["default"]().col("shapes", polygons).col("layerId", layerId).col("group", group).col("popup", popup).col("popupOptions", popupOptions).col("label", label).col("labelOptions", labelOptions).col("highlightOptions", highlightOptions).cbind(options);
    addLayers(this, "shape", df, function (df, i) {
      var shapes = df.get(i, "shapes");
      shapes = shapes.map(function (shape) {
        return _htmlwidgets2["default"].dataframeToD3(shape[0]);
      });

      if (shapes.length > 1) {
        return _leaflet2["default"].polyline(shapes, df.get(i));
      } else {
        return _leaflet2["default"].polyline(shapes[0], df.get(i));
      }
    });
  }
};

methods.removeMarker = function (layerId) {
  this.layerManager.removeLayer("marker", layerId);
};

methods.clearMarkers = function () {
  this.layerManager.clearLayers("marker");
};

methods.removeMarkerCluster = function (layerId) {
  this.layerManager.removeLayer("cluster", layerId);
};

methods.removeMarkerFromCluster = function (layerId, clusterId) {
  var cluster = this.layerManager.getLayer("cluster", clusterId);
  if (!cluster) return;
  cluster.clusterLayerStore.remove(layerId);
};

methods.clearMarkerClusters = function () {
  this.layerManager.clearLayers("cluster");
};

methods.removeShape = function (layerId) {
  this.layerManager.removeLayer("shape", layerId);
};

methods.clearShapes = function () {
  this.layerManager.clearLayers("shape");
};

methods.addRectangles = function (lat1, lng1, lat2, lng2, layerId, group, options, popup, popupOptions, label, labelOptions, highlightOptions) {
  var df = new _dataframe2["default"]().col("lat1", lat1).col("lng1", lng1).col("lat2", lat2).col("lng2", lng2).col("layerId", layerId).col("group", group).col("popup", popup).col("popupOptions", popupOptions).col("label", label).col("labelOptions", labelOptions).col("highlightOptions", highlightOptions).cbind(options);
  addLayers(this, "shape", df, function (df, i) {
    if (_jquery2["default"].isNumeric(df.get(i, "lat1")) && _jquery2["default"].isNumeric(df.get(i, "lng1")) && _jquery2["default"].isNumeric(df.get(i, "lat2")) && _jquery2["default"].isNumeric(df.get(i, "lng2"))) {
      return _leaflet2["default"].rectangle([[df.get(i, "lat1"), df.get(i, "lng1")], [df.get(i, "lat2"), df.get(i, "lng2")]], df.get(i));
    } else {
      return null;
    }
  });
};
/*
 * @param lat Array of arrays of latitude coordinates for polygons
 * @param lng Array of arrays of longitude coordinates for polygons
 */


methods.addPolygons = function (polygons, layerId, group, options, popup, popupOptions, label, labelOptions, highlightOptions) {
  if (polygons.length > 0) {
    var df = new _dataframe2["default"]().col("shapes", polygons).col("layerId", layerId).col("group", group).col("popup", popup).col("popupOptions", popupOptions).col("label", label).col("labelOptions", labelOptions).col("highlightOptions", highlightOptions).cbind(options);
    addLayers(this, "shape", df, function (df, i) {
      // This code used to use L.multiPolygon, but that caused
      // double-click on a multipolygon to fail to zoom in on the
      // map. Surprisingly, putting all the rings in a single
      // polygon seems to still work; complicated multipolygons
      // are still rendered correctly.
      var shapes = df.get(i, "shapes").map(function (polygon) {
        return polygon.map(_htmlwidgets2["default"].dataframeToD3);
      }).reduce(function (acc, val) {
        return acc.concat(val);
      }, []);
      return _leaflet2["default"].polygon(shapes, df.get(i));
    });
  }
};

methods.addGeoJSON = function (data, layerId, group, style) {
  // This time, self is actually needed because the callbacks below need
  // to access both the inner and outer senses of "this"
  var self = this;

  if (typeof data === "string") {
    data = JSON.parse(data);
  }

  var globalStyle = _jquery2["default"].extend({}, style, data.style || {});

  var gjlayer = _leaflet2["default"].geoJson(data, {
    style: function style(feature) {
      if (feature.style || feature.properties.style) {
        return _jquery2["default"].extend({}, globalStyle, feature.style, feature.properties.style);
      } else {
        return globalStyle;
      }
    },
    onEachFeature: function onEachFeature(feature, layer) {
      var extraInfo = {
        featureId: feature.id,
        properties: feature.properties
      };
      var popup = feature.properties ? feature.properties.popup : null;
      if (typeof popup !== "undefined" && popup !== null) layer.bindPopup(popup);
      layer.on("click", mouseHandler(self.id, layerId, group, "geojson_click", extraInfo), this);
      layer.on("mouseover", mouseHandler(self.id, layerId, group, "geojson_mouseover", extraInfo), this);
      layer.on("mouseout", mouseHandler(self.id, layerId, group, "geojson_mouseout", extraInfo), this);
    }
  });

  this.layerManager.addLayer(gjlayer, "geojson", layerId, group);
};

methods.removeGeoJSON = function (layerId) {
  this.layerManager.removeLayer("geojson", layerId);
};

methods.clearGeoJSON = function () {
  this.layerManager.clearLayers("geojson");
};

methods.addTopoJSON = function (data, layerId, group, style) {
  // This time, self is actually needed because the callbacks below need
  // to access both the inner and outer senses of "this"
  var self = this;

  if (typeof data === "string") {
    data = JSON.parse(data);
  }

  var globalStyle = _jquery2["default"].extend({}, style, data.style || {});

  var gjlayer = _leaflet2["default"].geoJson(null, {
    style: function style(feature) {
      if (feature.style || feature.properties.style) {
        return _jquery2["default"].extend({}, globalStyle, feature.style, feature.properties.style);
      } else {
        return globalStyle;
      }
    },
    onEachFeature: function onEachFeature(feature, layer) {
      var extraInfo = {
        featureId: feature.id,
        properties: feature.properties
      };
      var popup = feature.properties.popup;
      if (typeof popup !== "undefined" && popup !== null) layer.bindPopup(popup);
      layer.on("click", mouseHandler(self.id, layerId, group, "topojson_click", extraInfo), this);
      layer.on("mouseover", mouseHandler(self.id, layerId, group, "topojson_mouseover", extraInfo), this);
      layer.on("mouseout", mouseHandler(self.id, layerId, group, "topojson_mouseout", extraInfo), this);
    }
  });

  global.omnivore.topojson.parse(data, null, gjlayer);
  this.layerManager.addLayer(gjlayer, "topojson", layerId, group);
};

methods.removeTopoJSON = function (layerId) {
  this.layerManager.removeLayer("topojson", layerId);
};

methods.clearTopoJSON = function () {
  this.layerManager.clearLayers("topojson");
};

methods.addControl = function (html, position, layerId, classes) {
  function onAdd(map) {
    var div = _leaflet2["default"].DomUtil.create("div", classes);

    if (typeof layerId !== "undefined" && layerId !== null) {
      div.setAttribute("id", layerId);
    }

    this._div = div; // It's possible for window.Shiny to be true but Shiny.initializeInputs to
    // not be, when a static leaflet widget is included as part of the shiny
    // UI directly (not through leafletOutput or uiOutput). In this case we
    // don't do the normal Shiny stuff as that will all happen when Shiny
    // itself loads and binds the entire doc.

    if (window.Shiny && _shiny2["default"].initializeInputs) {
      _shiny2["default"].renderHtml(html, this._div);

      _shiny2["default"].initializeInputs(this._div);

      _shiny2["default"].bindAll(this._div);
    } else {
      this._div.innerHTML = html;
    }

    return this._div;
  }

  function onRemove(map) {
    if (window.Shiny && _shiny2["default"].unbindAll) {
      _shiny2["default"].unbindAll(this._div);
    }
  }

  var Control = _leaflet2["default"].Control.extend({
    options: {
      position: position
    },
    onAdd: onAdd,
    onRemove: onRemove
  });

  this.controls.add(new Control(), layerId, html);
};

methods.addCustomControl = function (control, layerId) {
  this.controls.add(control, layerId);
};

methods.removeControl = function (layerId) {
  this.controls.remove(layerId);
};

methods.getControl = function (layerId) {
  this.controls.get(layerId);
};

methods.clearControls = function () {
  this.controls.clear();
};

methods.addLegend = function (options) {
  var legend = _leaflet2["default"].control({
    position: options.position
  });

  var gradSpan;

  legend.onAdd = function (map) {
    var div = _leaflet2["default"].DomUtil.create("div", options.className),
        colors = options.colors,
        labels = options.labels,
        legendHTML = "";

    if (options.type === "numeric") {
      // # Formatting constants.
      var singleBinHeight = 20; // The distance between tick marks, in px

      var vMargin = 8; // If 1st tick mark starts at top of gradient, how
      // many extra px are needed for the top half of the
      // 1st label? (ditto for last tick mark/label)

      var tickWidth = 4; // How wide should tick marks be, in px?

      var labelPadding = 6; // How much distance to reserve for tick mark?
      // (Must be >= tickWidth)
      // # Derived formatting parameters.
      // What's the height of a single bin, in percentage (of gradient height)?
      // It might not just be 1/(n-1), if the gradient extends past the tick
      // marks (which can be the case for pretty cut points).

      var singleBinPct = (options.extra.p_n - options.extra.p_1) / (labels.length - 1); // Each bin is `singleBinHeight` high. How tall is the gradient?

      var totalHeight = 1 / singleBinPct * singleBinHeight + 1; // How far should the first tick be shifted down, relative to the top
      // of the gradient?

      var tickOffset = singleBinHeight / singleBinPct * options.extra.p_1;
      gradSpan = (0, _jquery2["default"])("<span/>").css({
        "background": "linear-gradient(" + colors + ")",
        "opacity": options.opacity,
        "height": totalHeight + "px",
        "width": "18px",
        "display": "block",
        "margin-top": vMargin + "px"
      });
      var leftDiv = (0, _jquery2["default"])("<div/>").css("float", "left"),
          rightDiv = (0, _jquery2["default"])("<div/>").css("float", "left");
      leftDiv.append(gradSpan);
      (0, _jquery2["default"])(div).append(leftDiv).append(rightDiv).append((0, _jquery2["default"])("<br>")); // Have to attach the div to the body at this early point, so that the
      // svg text getComputedTextLength() actually works, below.

      document.body.appendChild(div);
      var ns = "http://www.w3.org/2000/svg";
      var svg = document.createElementNS(ns, "svg");
      rightDiv.append(svg);
      var g = document.createElementNS(ns, "g");
      (0, _jquery2["default"])(g).attr("transform", "translate(0, " + vMargin + ")");
      svg.appendChild(g); // max label width needed to set width of svg, and right-justify text

      var maxLblWidth = 0; // Create tick marks and labels

      _jquery2["default"].each(labels, function (i, label) {
        var y = tickOffset + i * singleBinHeight + 0.5;
        var thisLabel = document.createElementNS(ns, "text");
        (0, _jquery2["default"])(thisLabel).text(labels[i]).attr("y", y).attr("dx", labelPadding).attr("dy", "0.5ex");
        g.appendChild(thisLabel);
        maxLblWidth = Math.max(maxLblWidth, thisLabel.getComputedTextLength());
        var thisTick = document.createElementNS(ns, "line");
        (0, _jquery2["default"])(thisTick).attr("x1", 0).attr("x2", tickWidth).attr("y1", y).attr("y2", y).attr("stroke-width", 1);
        g.appendChild(thisTick);
      }); // Now that we know the max label width, we can right-justify


      (0, _jquery2["default"])(svg).find("text").attr("dx", labelPadding + maxLblWidth).attr("text-anchor", "end"); // Final size for <svg>

      (0, _jquery2["default"])(svg).css({
        width: maxLblWidth + labelPadding + "px",
        height: totalHeight + vMargin * 2 + "px"
      });

      if (options.na_color && _jquery2["default"].inArray(options.na_label, labels) < 0) {
        (0, _jquery2["default"])(div).append("<div><i style=\"" + "background:" + options.na_color + ";opacity:" + options.opacity + ";margin-right:" + labelPadding + "px" + ";\"></i>" + options.na_label + "</div>");
      }
    } else {
      if (options.na_color && _jquery2["default"].inArray(options.na_label, labels) < 0) {
        colors.push(options.na_color);
        labels.push(options.na_label);
      }

      for (var i = 0; i < colors.length; i++) {
        legendHTML += "<i style=\"background:" + colors[i] + ";opacity:" + options.opacity + "\"></i> " + labels[i] + "<br>";
      }

      div.innerHTML = legendHTML;
    }

    if (options.title) (0, _jquery2["default"])(div).prepend("<div style=\"margin-bottom:3px\"><strong>" + options.title + "</strong></div>");
    return div;
  };

  if (options.group) {
    // Auto generate a layerID if not provided
    if (!options.layerId) {
      options.layerId = _leaflet2["default"].Util.stamp(legend);
    }

    var map = this;
    map.on("overlayadd", function (e) {
      if (e.name === options.group) {
        map.controls.add(legend, options.layerId);
      }
    });
    map.on("overlayremove", function (e) {
      if (e.name === options.group) {
        map.controls.remove(options.layerId);
      }
    });
    map.on("groupadd", function (e) {
      if (e.name === options.group) {
        map.controls.add(legend, options.layerId);
      }
    });
    map.on("groupremove", function (e) {
      if (e.name === options.group) {
        map.controls.remove(options.layerId);
      }
    });
  }

  this.controls.add(legend, options.layerId);
};

methods.addLayersControl = function (baseGroups, overlayGroups, options) {
  var _this4 = this;

  // Only allow one layers control at a time
  methods.removeLayersControl.call(this);
  var firstLayer = true;
  var base = {};

  _jquery2["default"].each((0, _util.asArray)(baseGroups), function (i, g) {
    var layer = _this4.layerManager.getLayerGroup(g, true);

    if (layer) {
      base[g] = layer; // Check if >1 base layers are visible; if so, hide all but the first one

      if (_this4.hasLayer(layer)) {
        if (firstLayer) {
          firstLayer = false;
        } else {
          _this4.removeLayer(layer);
        }
      }
    }
  });

  var overlay = {};

  _jquery2["default"].each((0, _util.asArray)(overlayGroups), function (i, g) {
    var layer = _this4.layerManager.getLayerGroup(g, true);

    if (layer) {
      overlay[g] = layer;
    }
  });

  this.currentLayersControl = _leaflet2["default"].control.layers(base, overlay, options);
  this.addControl(this.currentLayersControl);
};

methods.removeLayersControl = function () {
  if (this.currentLayersControl) {
    this.removeControl(this.currentLayersControl);
    this.currentLayersControl = null;
  }
};

methods.addScaleBar = function (options) {
  // Only allow one scale bar at a time
  methods.removeScaleBar.call(this);

  var scaleBar = _leaflet2["default"].control.scale(options).addTo(this);

  this.currentScaleBar = scaleBar;
};

methods.removeScaleBar = function () {
  if (this.currentScaleBar) {
    this.currentScaleBar.remove();
    this.currentScaleBar = null;
  }
};

methods.hideGroup = function (group) {
  var _this5 = this;

  _jquery2["default"].each((0, _util.asArray)(group), function (i, g) {
    var layer = _this5.layerManager.getLayerGroup(g, true);

    if (layer) {
      _this5.removeLayer(layer);
    }
  });
};

methods.showGroup = function (group) {
  var _this6 = this;

  _jquery2["default"].each((0, _util.asArray)(group), function (i, g) {
    var layer = _this6.layerManager.getLayerGroup(g, true);

    if (layer) {
      _this6.addLayer(layer);
    }
  });
};

function setupShowHideGroupsOnZoom(map) {
  if (map.leafletr._hasInitializedShowHideGroups) {
    return;
  }

  map.leafletr._hasInitializedShowHideGroups = true;

  function setVisibility(layer, visible, group) {
    if (visible !== map.hasLayer(layer)) {
      if (visible) {
        map.addLayer(layer);
        map.fire("groupadd", {
          "name": group,
          "layer": layer
        });
      } else {
        map.removeLayer(layer);
        map.fire("groupremove", {
          "name": group,
          "layer": layer
        });
      }
    }
  }

  function showHideGroupsOnZoom() {
    if (!map.layerManager) return;
    var zoom = map.getZoom();
    map.layerManager.getAllGroupNames().forEach(function (group) {
      var layer = map.layerManager.getLayerGroup(group, false);

      if (layer && typeof layer.zoomLevels !== "undefined") {
        setVisibility(layer, layer.zoomLevels === true || layer.zoomLevels.indexOf(zoom) >= 0, group);
      }
    });
  }

  map.showHideGroupsOnZoom = showHideGroupsOnZoom;
  map.on("zoomend", showHideGroupsOnZoom);
}

methods.setGroupOptions = function (group, options) {
  var _this7 = this;

  _jquery2["default"].each((0, _util.asArray)(group), function (i, g) {
    var layer = _this7.layerManager.getLayerGroup(g, true); // This slightly tortured check is because 0 is a valid value for zoomLevels


    if (typeof options.zoomLevels !== "undefined" && options.zoomLevels !== null) {
      layer.zoomLevels = (0, _util.asArray)(options.zoomLevels);
    }
  });

  setupShowHideGroupsOnZoom(this);
  this.showHideGroupsOnZoom();
};

methods.addRasterImage = function (uri, bounds, opacity, attribution, layerId, group) {
  // uri is a data URI containing an image. We want to paint this image as a
  // layer at (top-left) bounds[0] to (bottom-right) bounds[1].
  // We can't simply use ImageOverlay, as it uses bilinear scaling which looks
  // awful as you zoom in (and sometimes shifts positions or disappears).
  // Instead, we'll use a TileLayer.Canvas to draw pieces of the image.
  // First, some helper functions.
  // degree2tile converts latitude, longitude, and zoom to x and y tile
  // numbers. The tile numbers returned can be non-integral, as there's no
  // reason to expect that the lat/lng inputs are exactly on the border of two
  // tiles.
  //
  // We'll use this to convert the bounds we got from the server, into coords
  // in tile-space at a given zoom level. Note that once we do the conversion,
  // we don't to do any more trigonometry to convert between pixel coordinates
  // and tile coordinates; the source image pixel coords, destination canvas
  // pixel coords, and tile coords all can be scaled linearly.
  function degree2tile(lat, lng, zoom) {
    // See http://wiki.openstreetmap.org/wiki/Slippy_map_tilenames
    var latRad = lat * Math.PI / 180;
    var n = Math.pow(2, zoom);
    var x = (lng + 180) / 360 * n;
    var y = (1 - Math.log(Math.tan(latRad) + 1 / Math.cos(latRad)) / Math.PI) / 2 * n;
    return {
      x: x,
      y: y
    };
  } // Given a range [from,to) and either one or two numbers, returns true if
  // there is any overlap between [x,x1) and the range--or if x1 is omitted,
  // then returns true if x is within [from,to).


  function overlap(from, to, x,
  /* optional */
  x1) {
    if (arguments.length == 3) x1 = x;
    return x < to && x1 >= from;
  }

  function getCanvasSmoothingProperty(ctx) {
    var candidates = ["imageSmoothingEnabled", "mozImageSmoothingEnabled", "webkitImageSmoothingEnabled", "msImageSmoothingEnabled"];

    for (var i = 0; i < candidates.length; i++) {
      if (typeof ctx[candidates[i]] !== "undefined") {
        return candidates[i];
      }
    }

    return null;
  } // Our general strategy is to:
  // 1. Load the data URI in an Image() object, so we can get its pixel
  //    dimensions and the underlying image data. (We could have done this
  //    by not encoding as PNG at all but just send an array of RGBA values
  //    from the server, but that would inflate the JSON too much.)
  // 2. Create a hidden canvas that we use just to extract the image data
  //    from the Image (using Context2D.getImageData()).
  // 3. Create a TileLayer.Canvas and add it to the map.
  // We want to synchronously create and attach the TileLayer.Canvas (so an
  // immediate call to clearRasters() will be respected, for example), but
  // Image loads its data asynchronously. Fortunately we can resolve this
  // by putting TileLayer.Canvas into async mode, which will let us create
  // and attach the layer but have it wait until the image is loaded before
  // it actually draws anything.
  // These are the variables that we will populate once the image is loaded.


  var imgData = null; // 1d row-major array, four [0-255] integers per pixel

  var imgDataMipMapper = null;
  var w = null; // image width in pixels

  var h = null; // image height in pixels
  // We'll use this array to store callbacks that need to be invoked once
  // imgData, w, and h have been resolved.

  var imgDataCallbacks = []; // Consumers of imgData, w, and h can call this to be notified when data
  // is available.

  function getImageData(callback) {
    if (imgData != null) {
      // Must not invoke the callback immediately; it's too confusing and
      // fragile to have a function invoke the callback *either* immediately
      // or in the future. Better to be consistent here.
      setTimeout(function () {
        callback(imgData, w, h, imgDataMipMapper);
      }, 0);
    } else {
      imgDataCallbacks.push(callback);
    }
  }

  var img = new Image();

  img.onload = function () {
    // Save size
    w = img.width;
    h = img.height; // Create a dummy canvas to extract the image data

    var imgDataCanvas = document.createElement("canvas");
    imgDataCanvas.width = w;
    imgDataCanvas.height = h;
    imgDataCanvas.style.display = "none";
    document.body.appendChild(imgDataCanvas);
    var imgDataCtx = imgDataCanvas.getContext("2d");
    imgDataCtx.drawImage(img, 0, 0); // Save the image data.

    imgData = imgDataCtx.getImageData(0, 0, w, h).data;
    imgDataMipMapper = new _mipmapper2["default"](img); // Done with the canvas, remove it from the page so it can be gc'd.

    document.body.removeChild(imgDataCanvas); // Alert any getImageData callers who are waiting.

    for (var i = 0; i < imgDataCallbacks.length; i++) {
      imgDataCallbacks[i](imgData, w, h, imgDataMipMapper);
    }

    imgDataCallbacks = [];
  };

  img.src = uri;

  var canvasTiles = _leaflet2["default"].gridLayer({
    opacity: opacity,
    attribution: attribution,
    detectRetina: true,
    async: true
  }); // NOTE: The done() function MUST NOT be invoked until after the current
  // tick; done() looks in Leaflet's tile cache for the current tile, and
  // since it's still being constructed, it won't be found.


  canvasTiles.createTile = function (tilePoint, done) {
    var zoom = tilePoint.z;

    var canvas = _leaflet2["default"].DomUtil.create("canvas");

    var error; // setup tile width and height according to the options

    var size = this.getTileSize();
    canvas.width = size.x;
    canvas.height = size.y;
    getImageData(function (imgData, w, h, mipmapper) {
      try {
        // The Context2D we'll being drawing onto. It's always 256x256.
        var ctx = canvas.getContext("2d"); // Convert our image data's top-left and bottom-right locations into
        // x/y tile coordinates. This is essentially doing a spherical mercator
        // projection, then multiplying by 2^zoom.

        var topLeft = degree2tile(bounds[0][0], bounds[0][1], zoom);
        var bottomRight = degree2tile(bounds[1][0], bounds[1][1], zoom); // The size of the image in x/y tile coordinates.

        var extent = {
          x: bottomRight.x - topLeft.x,
          y: bottomRight.y - topLeft.y
        }; // Short circuit if tile is totally disjoint from image.

        if (!overlap(tilePoint.x, tilePoint.x + 1, topLeft.x, bottomRight.x)) return;
        if (!overlap(tilePoint.y, tilePoint.y + 1, topLeft.y, bottomRight.y)) return; // The linear resolution of the tile we're drawing is always 256px per tile unit.
        // If the linear resolution (in either direction) of the image is less than 256px
        // per tile unit, then use nearest neighbor; otherwise, use the canvas's built-in
        // scaling.

        var imgRes = {
          x: w / extent.x,
          y: h / extent.y
        }; // We can do the actual drawing in one of three ways:
        // - Call drawImage(). This is easy and fast, and results in smooth
        //   interpolation (bilinear?). This is what we want when we are
        //   reducing the image from its native size.
        // - Call drawImage() with imageSmoothingEnabled=false. This is easy
        //   and fast and gives us nearest-neighbor interpolation, which is what
        //   we want when enlarging the image. However, it's unsupported on many
        //   browsers (including QtWebkit).
        // - Do a manual nearest-neighbor interpolation. This is what we'll fall
        //   back to when enlarging, and imageSmoothingEnabled isn't supported.
        //   In theory it's slower, but still pretty fast on my machine, and the
        //   results look the same AFAICT.
        // Is imageSmoothingEnabled supported? If so, we can let canvas do
        // nearest-neighbor interpolation for us.

        var smoothingProperty = getCanvasSmoothingProperty(ctx);

        if (smoothingProperty || imgRes.x >= 256 && imgRes.y >= 256) {
          // Use built-in scaling
          // Turn off anti-aliasing if necessary
          if (smoothingProperty) {
            ctx[smoothingProperty] = imgRes.x >= 256 && imgRes.y >= 256;
          } // Don't necessarily draw with the full-size image; if we're
          // downscaling, use the mipmapper to get a pre-downscaled image
          // (see comments on Mipmapper class for why this matters).


          mipmapper.getBySize(extent.x * 256, extent.y * 256, function (mip) {
            // It's possible that the image will go off the edge of the canvas--
            // that's OK, the canvas should clip appropriately.
            ctx.drawImage(mip, // Convert abs tile coords to rel tile coords, then *256 to convert
            // to rel pixel coords
            (topLeft.x - tilePoint.x) * 256, (topLeft.y - tilePoint.y) * 256, // Always draw the whole thing and let canvas clip; so we can just
            // convert from size in tile coords straight to pixels
            extent.x * 256, extent.y * 256);
          });
        } else {
          // Use manual nearest-neighbor interpolation
          // Calculate the source image pixel coordinates that correspond with
          // the top-left and bottom-right of this tile. (If the source image
          // only partially overlaps the tile, we use max/min to limit the
          // sourceStart/End to only reflect the overlapping portion.)
          var sourceStart = {
            x: Math.max(0, Math.floor((tilePoint.x - topLeft.x) * imgRes.x)),
            y: Math.max(0, Math.floor((tilePoint.y - topLeft.y) * imgRes.y))
          };
          var sourceEnd = {
            x: Math.min(w, Math.ceil((tilePoint.x + 1 - topLeft.x) * imgRes.x)),
            y: Math.min(h, Math.ceil((tilePoint.y + 1 - topLeft.y) * imgRes.y))
          }; // The size, in dest pixels, that each source pixel should occupy.
          // This might be greater or less than 1 (e.g. if x and y resolution
          // are very different).

          var pixelSize = {
            x: 256 / imgRes.x,
            y: 256 / imgRes.y
          }; // For each pixel in the source image that overlaps the tile...

          for (var row = sourceStart.y; row < sourceEnd.y; row++) {
            for (var col = sourceStart.x; col < sourceEnd.x; col++) {
              // ...extract the pixel data...
              var i = (row * w + col) * 4;
              var r = imgData[i];
              var g = imgData[i + 1];
              var b = imgData[i + 2];
              var a = imgData[i + 3];
              ctx.fillStyle = "rgba(" + [r, g, b, a / 255].join(",") + ")"; // ...calculate the corresponding pixel coord in the dest image
              // where it should be drawn...

              var pixelPos = {
                x: (col / imgRes.x + topLeft.x - tilePoint.x) * 256,
                y: (row / imgRes.y + topLeft.y - tilePoint.y) * 256
              }; // ...and draw a rectangle there.

              ctx.fillRect(Math.round(pixelPos.x), Math.round(pixelPos.y), // Looks crazy, but this is necessary to prevent rounding from
              // causing overlap between this rect and its neighbors. The
              // minuend is the location of the next pixel, while the
              // subtrahend is the position of the current pixel (to turn an
              // absolute coordinate to a width/height). Yes, I had to look
              // up minuend and subtrahend.
              Math.round(pixelPos.x + pixelSize.x) - Math.round(pixelPos.x), Math.round(pixelPos.y + pixelSize.y) - Math.round(pixelPos.y));
            }
          }
        }
      } catch (e) {
        error = e;
      } finally {
        done(error, canvas);
      }
    });
    return canvas;
  };

  this.layerManager.addLayer(canvasTiles, "image", layerId, group);
};

methods.removeImage = function (layerId) {
  this.layerManager.removeLayer("image", layerId);
};

methods.clearImages = function () {
  this.layerManager.clearLayers("image");
};

methods.addMeasure = function (options) {
  // if a measureControl already exists, then remove it and
  //   replace with a new one
  methods.removeMeasure.call(this);
  this.measureControl = _leaflet2["default"].control.measure(options);
  this.addControl(this.measureControl);
};

methods.removeMeasure = function () {
  if (this.measureControl) {
    this.removeControl(this.measureControl);
    this.measureControl = null;
  }
};

methods.addSelect = function (ctGroup) {
  var _this8 = this;

  methods.removeSelect.call(this);
  this._selectButton = _leaflet2["default"].easyButton({
    states: [{
      stateName: "select-inactive",
      icon: "ion-qr-scanner",
      title: "Make a selection",
      onClick: function onClick(btn, map) {
        btn.state("select-active");
        _this8._locationFilter = new _leaflet2["default"].LocationFilter2();

        if (ctGroup) {
          var selectionHandle = new global.crosstalk.SelectionHandle(ctGroup);
          selectionHandle.on("change", function (e) {
            if (e.sender !== selectionHandle) {
              if (_this8._locationFilter) {
                _this8._locationFilter.disable();

                btn.state("select-inactive");
              }
            }
          });

          var handler = function handler(e) {
            _this8.layerManager.brush(_this8._locationFilter.getBounds(), {
              sender: selectionHandle
            });
          };

          _this8._locationFilter.on("enabled", handler);

          _this8._locationFilter.on("change", handler);

          _this8._locationFilter.on("disabled", function () {
            selectionHandle.close();
            _this8._locationFilter = null;
          });
        }

        _this8._locationFilter.addTo(map);
      }
    }, {
      stateName: "select-active",
      icon: "ion-close-round",
      title: "Dismiss selection",
      onClick: function onClick(btn, map) {
        btn.state("select-inactive");

        _this8._locationFilter.disable(); // If explicitly dismissed, clear the crosstalk selections


        _this8.layerManager.unbrush();
      }
    }]
  });

  this._selectButton.addTo(this);
};

methods.removeSelect = function () {
  if (this._locationFilter) {
    this._locationFilter.disable();
  }

  if (this._selectButton) {
    this.removeControl(this._selectButton);
    this._selectButton = null;
  }
};

methods.createMapPane = function (name, zIndex) {
  this.createPane(name);
  this.getPane(name).style.zIndex = zIndex;
};


}).call(this)}).call(this,typeof global !== "undefined" ? global : typeof self !== "undefined" ? self : typeof window !== "undefined" ? window : {})
},{"./cluster-layer-store":1,"./crs_utils":3,"./dataframe":4,"./global/htmlwidgets":8,"./global/jquery":9,"./global/leaflet":10,"./global/shiny":12,"./mipmapper":16,"./util":17}],16:[function(require,module,exports){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});

function _classCallCheck(instance, Constructor) { if (!(instance instanceof Constructor)) { throw new TypeError("Cannot call a class as a function"); } }

function _defineProperties(target, props) { for (var i = 0; i < props.length; i++) { var descriptor = props[i]; descriptor.enumerable = descriptor.enumerable || false; descriptor.configurable = true; if ("value" in descriptor) descriptor.writable = true; Object.defineProperty(target, descriptor.key, descriptor); } }

function _createClass(Constructor, protoProps, staticProps) { if (protoProps) _defineProperties(Constructor.prototype, protoProps); if (staticProps) _defineProperties(Constructor, staticProps); return Constructor; }

// This class simulates a mipmap, which shrinks images by powers of two. This
// stepwise reduction results in "pixel-perfect downscaling" (where every
// pixel of the original image has some contribution to the downscaled image)
// as opposed to a single-step downscaling which will discard a lot of data
// (and with sparse images at small scales can give very surprising results).
var Mipmapper = /*#__PURE__*/function () {
  function Mipmapper(img) {
    _classCallCheck(this, Mipmapper);

    this._layers = [img];
  } // The various functions on this class take a callback function BUT MAY OR MAY
  // NOT actually behave asynchronously.


  _createClass(Mipmapper, [{
    key: "getBySize",
    value: function getBySize(desiredWidth, desiredHeight, callback) {
      var _this = this;

      var i = 0;
      var lastImg = this._layers[0];

      var testNext = function testNext() {
        _this.getByIndex(i, function (img) {
          // If current image is invalid (i.e. too small to be rendered) or
          // it's smaller than what we wanted, return the last known good image.
          if (!img || img.width < desiredWidth || img.height < desiredHeight) {
            callback(lastImg);
            return;
          } else {
            lastImg = img;
            i++;
            testNext();
            return;
          }
        });
      };

      testNext();
    }
  }, {
    key: "getByIndex",
    value: function getByIndex(i, callback) {
      var _this2 = this;

      if (this._layers[i]) {
        callback(this._layers[i]);
        return;
      }

      this.getByIndex(i - 1, function (prevImg) {
        if (!prevImg) {
          // prevImg could not be calculated (too small, possibly)
          callback(null);
          return;
        }

        if (prevImg.width < 2 || prevImg.height < 2) {
          // Can't reduce this image any further
          callback(null);
          return;
        } // If reduce ever becomes truly asynchronous, we should stuff a promise or
        // something into this._layers[i] before calling this.reduce(), to prevent
        // redundant reduce operations from happening.


        _this2.reduce(prevImg, function (reducedImg) {
          _this2._layers[i] = reducedImg;
          callback(reducedImg);
          return;
        });
      });
    }
  }, {
    key: "reduce",
    value: function reduce(img, callback) {
      var imgDataCanvas = document.createElement("canvas");
      imgDataCanvas.width = Math.ceil(img.width / 2);
      imgDataCanvas.height = Math.ceil(img.height / 2);
      imgDataCanvas.style.display = "none";
      document.body.appendChild(imgDataCanvas);

      try {
        var imgDataCtx = imgDataCanvas.getContext("2d");
        imgDataCtx.drawImage(img, 0, 0, img.width / 2, img.height / 2);
        callback(imgDataCanvas);
      } finally {
        document.body.removeChild(imgDataCanvas);
      }
    }
  }]);

  return Mipmapper;
}();

exports["default"] = Mipmapper;


},{}],17:[function(require,module,exports){
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports.log = log;
exports.recycle = recycle;
exports.asArray = asArray;

function log(message) {
  /* eslint-disable no-console */
  if (console && console.log) console.log(message);
  /* eslint-enable no-console */
}

function recycle(values, length, inPlace) {
  if (length === 0 && !inPlace) return [];

  if (!(values instanceof Array)) {
    if (inPlace) {
      throw new Error("Can't do in-place recycling of a non-Array value");
    }

    values = [values];
  }

  if (typeof length === "undefined") length = values.length;
  var dest = inPlace ? values : [];
  var origLength = values.length;

  while (dest.length < length) {
    dest.push(values[dest.length % origLength]);
  }

  if (dest.length > length) {
    dest.splice(length, dest.length - length);
  }

  return dest;
}

function asArray(value) {
  if (value instanceof Array) return value;else return [value];
}


},{}]},{},[13]);
</script>
<script>(function (root, factory) {
	if (typeof define === 'function' && define.amd) {
		// AMD. Register as an anonymous module.
		define(['leaflet'], factory);
	} else if (typeof modules === 'object' && module.exports) {
		// define a Common JS module that relies on 'leaflet'
		module.exports = factory(require('leaflet'));
	} else {
		// Assume Leaflet is loaded into global object L already
		factory(L);
	}
}(this, function (L) {
	'use strict';

	L.TileLayer.Provider = L.TileLayer.extend({
		initialize: function (arg, options) {
			var providers = L.TileLayer.Provider.providers;

			var parts = arg.split('.');

			var providerName = parts[0];
			var variantName = parts[1];

			if (!providers[providerName]) {
				throw 'No such provider (' + providerName + ')';
			}

			var provider = {
				url: providers[providerName].url,
				options: providers[providerName].options
			};

			// overwrite values in provider from variant.
			if (variantName && 'variants' in providers[providerName]) {
				if (!(variantName in providers[providerName].variants)) {
					throw 'No such variant of ' + providerName + ' (' + variantName + ')';
				}
				var variant = providers[providerName].variants[variantName];
				var variantOptions;
				if (typeof variant === 'string') {
					variantOptions = {
						variant: variant
					};
				} else {
					variantOptions = variant.options;
				}
				provider = {
					url: variant.url || provider.url,
					options: L.Util.extend({}, provider.options, variantOptions)
				};
			}

			// replace attribution placeholders with their values from toplevel provider attribution,
			// recursively
			var attributionReplacer = function (attr) {
				if (attr.indexOf('{attribution.') === -1) {
					return attr;
				}
				return attr.replace(/\{attribution.(\w*)\}/g,
					function (match, attributionName) {
						return attributionReplacer(providers[attributionName].options.attribution);
					}
				);
			};
			provider.options.attribution = attributionReplacer(provider.options.attribution);

			// Compute final options combining provider options with any user overrides
			var layerOpts = L.Util.extend({}, provider.options, options);
			L.TileLayer.prototype.initialize.call(this, provider.url, layerOpts);
		}
	});

	/**
	 * Definition of providers.
	 * see http://leafletjs.com/reference.html#tilelayer for options in the options map.
	 */

	L.TileLayer.Provider.providers = {
		OpenStreetMap: {
			url: 'https://tile.openstreetmap.org/{z}/{x}/{y}.png',
			options: {
				maxZoom: 19,
				attribution:
					'&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
			},
			variants: {
				Mapnik: {},
				DE: {
					url: 'https://tile.openstreetmap.de/{z}/{x}/{y}.png',
					options: {
						maxZoom: 18
					}
				},
				CH: {
					url: 'https://tile.osm.ch/switzerland/{z}/{x}/{y}.png',
					options: {
						maxZoom: 18,
						bounds: [[45, 5], [48, 11]]
					}
				},
				France: {
					url: 'https://{s}.tile.openstreetmap.fr/osmfr/{z}/{x}/{y}.png',
					options: {
						maxZoom: 20,
						attribution: '&copy; OpenStreetMap France | {attribution.OpenStreetMap}'
					}
				},
				HOT: {
					url: 'https://{s}.tile.openstreetmap.fr/hot/{z}/{x}/{y}.png',
					options: {
						attribution:
							'{attribution.OpenStreetMap}, ' +
							'Tiles style by <a href="https://www.hotosm.org/" target="_blank">Humanitarian OpenStreetMap Team</a> ' +
							'hosted by <a href="https://openstreetmap.fr/" target="_blank">OpenStreetMap France</a>'
					}
				},
				BZH: {
					url: 'https://tile.openstreetmap.bzh/br/{z}/{x}/{y}.png',
					options: {
						attribution: '{attribution.OpenStreetMap}, Tiles courtesy of <a href="http://www.openstreetmap.bzh/" target="_blank">Breton OpenStreetMap Team</a>',
						bounds: [[46.2, -5.5], [50, 0.7]]
					}
				}
			}
		},
		MapTilesAPI: {
			url: 'https://maptiles.p.rapidapi.com/{variant}/{z}/{x}/{y}.png?rapidapi-key={apikey}',
			options: {
				attribution:
					'&copy; <a href="http://www.maptilesapi.com/">MapTiles API</a>, {attribution.OpenStreetMap}',
				variant: 'en/map/v1',
				// Get your own MapTiles API access token here : https://www.maptilesapi.com/
				// NB : this is a demonstration key that comes with no guarantee and not to be used in production
				apikey: '<insert your api key here>',
				maxZoom: 19
			},
			variants: {
				OSMEnglish: {
					options: {
						variant: 'en/map/v1'
					}
				},
				OSMFrancais: {
					options: {
						variant: 'fr/map/v1'
					}
				},
				OSMEspagnol: {
					options: {
						variant: 'es/map/v1'
					}
				}
			}
		},
		OpenSeaMap: {
			url: 'https://tiles.openseamap.org/seamark/{z}/{x}/{y}.png',
			options: {
				attribution: 'Map data: &copy; <a href="http://www.openseamap.org">OpenSeaMap</a> contributors'
			}
		},
		OPNVKarte: {
			url: 'https://tileserver.memomaps.de/tilegen/{z}/{x}/{y}.png',
			options: {
				maxZoom: 18,
				attribution: 'Map <a href="https://memomaps.de/">memomaps.de</a> <a href="http://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>, map data {attribution.OpenStreetMap}'
			}
		},
		OpenTopoMap: {
			url: 'https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png',
			options: {
				maxZoom: 17,
				attribution: 'Map data: {attribution.OpenStreetMap}, <a href="http://viewfinderpanoramas.org">SRTM</a> | Map style: &copy; <a href="https://opentopomap.org">OpenTopoMap</a> (<a href="https://creativecommons.org/licenses/by-sa/3.0/">CC-BY-SA</a>)'
			}
		},
		OpenRailwayMap: {
			url: 'https://{s}.tiles.openrailwaymap.org/standard/{z}/{x}/{y}.png',
			options: {
				maxZoom: 19,
				attribution: 'Map data: {attribution.OpenStreetMap} | Map style: &copy; <a href="https://www.OpenRailwayMap.org">OpenRailwayMap</a> (<a href="https://creativecommons.org/licenses/by-sa/3.0/">CC-BY-SA</a>)'
			}
		},
		OpenFireMap: {
			url: 'http://openfiremap.org/hytiles/{z}/{x}/{y}.png',
			options: {
				maxZoom: 19,
				attribution: 'Map data: {attribution.OpenStreetMap} | Map style: &copy; <a href="http://www.openfiremap.org">OpenFireMap</a> (<a href="https://creativecommons.org/licenses/by-sa/3.0/">CC-BY-SA</a>)'
			}
		},
		SafeCast: {
			url: 'https://s3.amazonaws.com/te512.safecast.org/{z}/{x}/{y}.png',
			options: {
				maxZoom: 16,
				attribution: 'Map data: {attribution.OpenStreetMap} | Map style: &copy; <a href="https://blog.safecast.org/about/">SafeCast</a> (<a href="https://creativecommons.org/licenses/by-sa/3.0/">CC-BY-SA</a>)'
			}
		},
		Stadia: {
			url: 'https://tiles.stadiamaps.com/tiles/{variant}/{z}/{x}/{y}{r}.{ext}',
			options: {
				minZoom: 0,
				maxZoom: 20,
				attribution:
					'&copy; <a href="https://www.stadiamaps.com/" target="_blank">Stadia Maps</a> ' +
					'&copy; <a href="https://openmaptiles.org/" target="_blank">OpenMapTiles</a> ' +
					'{attribution.OpenStreetMap}',
				variant: 'alidade_smooth',
				ext: 'png'
			},
			variants: {
				AlidadeSmooth: 'alidade_smooth',
				AlidadeSmoothDark: 'alidade_smooth_dark',
				OSMBright: 'osm_bright',
				Outdoors: 'outdoors',
				StamenToner: {
					options: {
						attribution:
							'&copy; <a href="https://www.stadiamaps.com/" target="_blank">Stadia Maps</a> ' +
							'&copy; <a href="https://www.stamen.com/" target="_blank">Stamen Design</a> ' +
							'&copy; <a href="https://openmaptiles.org/" target="_blank">OpenMapTiles</a> ' +
							'{attribution.OpenStreetMap}',
						variant: 'stamen_toner'
					}
				},
				StamenTonerBackground: {
					options: {
						attribution:
							'&copy; <a href="https://www.stadiamaps.com/" target="_blank">Stadia Maps</a> ' +
							'&copy; <a href="https://www.stamen.com/" target="_blank">Stamen Design</a> ' +
							'&copy; <a href="https://openmaptiles.org/" target="_blank">OpenMapTiles</a> ' +
							'{attribution.OpenStreetMap}',
						variant: 'stamen_toner_background'
					}
				},
				StamenTonerLines: {
					options: {
						attribution:
							'&copy; <a href="https://www.stadiamaps.com/" target="_blank">Stadia Maps</a> ' +
							'&copy; <a href="https://www.stamen.com/" target="_blank">Stamen Design</a> ' +
							'&copy; <a href="https://openmaptiles.org/" target="_blank">OpenMapTiles</a> ' +
							'{attribution.OpenStreetMap}',
						variant: 'stamen_toner_lines'
					}
				},
				StamenTonerLabels: {
					options: {
						attribution:
							'&copy; <a href="https://www.stadiamaps.com/" target="_blank">Stadia Maps</a> ' +
							'&copy; <a href="https://www.stamen.com/" target="_blank">Stamen Design</a> ' +
							'&copy; <a href="https://openmaptiles.org/" target="_blank">OpenMapTiles</a> ' +
							'{attribution.OpenStreetMap}',
						variant: 'stamen_toner_labels'
					}
				},
				StamenTonerLite: {
					options: {
						attribution:
							'&copy; <a href="https://www.stadiamaps.com/" target="_blank">Stadia Maps</a> ' +
							'&copy; <a href="https://www.stamen.com/" target="_blank">Stamen Design</a> ' +
							'&copy; <a href="https://openmaptiles.org/" target="_blank">OpenMapTiles</a> ' +
							'{attribution.OpenStreetMap}',
						variant: 'stamen_toner_lite'
					}
				},
				StamenWatercolor: {
					url: 'https://tiles.stadiamaps.com/tiles/{variant}/{z}/{x}/{y}.{ext}',
					options: {
						attribution:
							'&copy; <a href="https://www.stadiamaps.com/" target="_blank">Stadia Maps</a> ' +
							'&copy; <a href="https://www.stamen.com/" target="_blank">Stamen Design</a> ' +
							'&copy; <a href="https://openmaptiles.org/" target="_blank">OpenMapTiles</a> ' +
							'{attribution.OpenStreetMap}',
						variant: 'stamen_watercolor',
						ext: 'jpg',
						minZoom: 1,
						maxZoom: 16
					}
				},
				StamenTerrain: {
					options: {
						attribution:
							'&copy; <a href="https://www.stadiamaps.com/" target="_blank">Stadia Maps</a> ' +
							'&copy; <a href="https://www.stamen.com/" target="_blank">Stamen Design</a> ' +
							'&copy; <a href="https://openmaptiles.org/" target="_blank">OpenMapTiles</a> ' +
							'{attribution.OpenStreetMap}',
						variant: 'stamen_terrain',
						minZoom: 0,
						maxZoom: 18
					}
				},
				StamenTerrainBackground: {
					options: {
						attribution:
							'&copy; <a href="https://www.stadiamaps.com/" target="_blank">Stadia Maps</a> ' +
							'&copy; <a href="https://www.stamen.com/" target="_blank">Stamen Design</a> ' +
							'&copy; <a href="https://openmaptiles.org/" target="_blank">OpenMapTiles</a> ' +
							'{attribution.OpenStreetMap}',
						variant: 'stamen_terrain_background',
						minZoom: 0,
						maxZoom: 18
					}
				},
				StamenTerrainLabels: {
					options: {
						attribution:
							'&copy; <a href="https://www.stadiamaps.com/" target="_blank">Stadia Maps</a> ' +
							'&copy; <a href="https://www.stamen.com/" target="_blank">Stamen Design</a> ' +
							'&copy; <a href="https://openmaptiles.org/" target="_blank">OpenMapTiles</a> ' +
							'{attribution.OpenStreetMap}',
						variant: 'stamen_terrain_labels',
						minZoom: 0,
						maxZoom: 18
					}
				},
				StamenTerrainLines: {
					options: {
						attribution:
							'&copy; <a href="https://www.stadiamaps.com/" target="_blank">Stadia Maps</a> ' +
							'&copy; <a href="https://www.stamen.com/" target="_blank">Stamen Design</a> ' +
							'&copy; <a href="https://openmaptiles.org/" target="_blank">OpenMapTiles</a> ' +
							'{attribution.OpenStreetMap}',
						variant: 'stamen_terrain_lines',
						minZoom: 0,
						maxZoom: 18
					}
				}
			}
		},
		Thunderforest: {
			url: 'https://{s}.tile.thunderforest.com/{variant}/{z}/{x}/{y}.png?apikey={apikey}',
			options: {
				attribution:
					'&copy; <a href="http://www.thunderforest.com/">Thunderforest</a>, {attribution.OpenStreetMap}',
				variant: 'cycle',
				apikey: '<insert your api key here>',
				maxZoom: 22
			},
			variants: {
				OpenCycleMap: 'cycle',
				Transport: {
					options: {
						variant: 'transport'
					}
				},
				TransportDark: {
					options: {
						variant: 'transport-dark'
					}
				},
				SpinalMap: {
					options: {
						variant: 'spinal-map'
					}
				},
				Landscape: 'landscape',
				Outdoors: 'outdoors',
				Pioneer: 'pioneer',
				MobileAtlas: 'mobile-atlas',
				Neighbourhood: 'neighbourhood'
			}
		},
		CyclOSM: {
			url: 'https://{s}.tile-cyclosm.openstreetmap.fr/cyclosm/{z}/{x}/{y}.png',
			options: {
				maxZoom: 20,
				attribution: '<a href="https://github.com/cyclosm/cyclosm-cartocss-style/releases" title="CyclOSM - Open Bicycle render">CyclOSM</a> | Map data: {attribution.OpenStreetMap}'
			}
		},
		Jawg: {
			url: 'https://{s}.tile.jawg.io/{variant}/{z}/{x}/{y}{r}.png?access-token={accessToken}',
			options: {
				attribution:
					'<a href="http://jawg.io" title="Tiles Courtesy of Jawg Maps" target="_blank">&copy; <b>Jawg</b>Maps</a> ' +
					'{attribution.OpenStreetMap}',
				minZoom: 0,
				maxZoom: 22,
				subdomains: 'abcd',
				variant: 'jawg-terrain',
				// Get your own Jawg access token here : https://www.jawg.io/lab/
				// NB : this is a demonstration key that comes with no guarantee
				accessToken: '<insert your access token here>',
			},
			variants: {
				Streets: 'jawg-streets',
				Terrain: 'jawg-terrain',
				Sunny: 'jawg-sunny',
				Dark: 'jawg-dark',
				Light: 'jawg-light',
				Matrix: 'jawg-matrix'
			}
		},
		MapBox: {
			url: 'https://api.mapbox.com/styles/v1/{id}/tiles/{z}/{x}/{y}{r}?access_token={accessToken}',
			options: {
				attribution:
					'&copy; <a href="https://www.mapbox.com/about/maps/" target="_blank">Mapbox</a> ' +
					'{attribution.OpenStreetMap} ' +
					'<a href="https://www.mapbox.com/map-feedback/" target="_blank">Improve this map</a>',
				tileSize: 512,
				maxZoom: 18,
				zoomOffset: -1,
				id: 'mapbox/streets-v11',
				accessToken: '<insert your access token here>',
			}
		},
		MapTiler: {
			url: 'https://api.maptiler.com/maps/{variant}/{z}/{x}/{y}{r}.{ext}?key={key}',
			options: {
				attribution:
					'<a href="https://www.maptiler.com/copyright/" target="_blank">&copy; MapTiler</a> <a href="https://www.openstreetmap.org/copyright" target="_blank">&copy; OpenStreetMap contributors</a>',
				variant: 'streets',
				ext: 'png',
				key: '<insert your MapTiler Cloud API key here>',
				tileSize: 512,
				zoomOffset: -1,
				minZoom: 0,
				maxZoom: 21
			},
			variants: {
				Streets: 'streets',
				Basic: 'basic',
				Bright: 'bright',
				Pastel: 'pastel',
				Positron: 'positron',
				Hybrid: {
					options: {
						variant: 'hybrid',
						ext: 'jpg'
					}
				},
				Toner: 'toner',
				Topo: 'topo',
				Voyager: 'voyager'
			}
		},
		TomTom: {
			url: 'https://{s}.api.tomtom.com/map/1/tile/{variant}/{style}/{z}/{x}/{y}.{ext}?key={apikey}',
			options: {
				variant: 'basic',
				maxZoom: 22,
				attribution:
					'<a href="https://tomtom.com" target="_blank">&copy;  1992 - ' + new Date().getFullYear() + ' TomTom.</a> ',
				subdomains: 'abcd',
				style: 'main',
				ext: 'png',
				apikey: '<insert your API key here>',
			},
			variants: {
				Basic: 'basic',
				Hybrid: 'hybrid',
				Labels: 'labels'
			}
		},
		Esri: {
			url: 'https://server.arcgisonline.com/ArcGIS/rest/services/{variant}/MapServer/tile/{z}/{y}/{x}',
			options: {
				variant: 'World_Street_Map',
				attribution: 'Tiles &copy; Esri'
			},
			variants: {
				WorldStreetMap: {
					options: {
						attribution:
							'{attribution.Esri} &mdash; ' +
							'Source: Esri, DeLorme, NAVTEQ, USGS, Intermap, iPC, NRCAN, Esri Japan, METI, Esri China (Hong Kong), Esri (Thailand), TomTom, 2012'
					}
				},
				DeLorme: {
					options: {
						variant: 'Specialty/DeLorme_World_Base_Map',
						minZoom: 1,
						maxZoom: 11,
						attribution: '{attribution.Esri} &mdash; Copyright: &copy;2012 DeLorme'
					}
				},
				WorldTopoMap: {
					options: {
						variant: 'World_Topo_Map',
						attribution:
							'{attribution.Esri} &mdash; ' +
							'Esri, DeLorme, NAVTEQ, TomTom, Intermap, iPC, USGS, FAO, NPS, NRCAN, GeoBase, Kadaster NL, Ordnance Survey, Esri Japan, METI, Esri China (Hong Kong), and the GIS User Community'
					}
				},
				WorldImagery: {
					options: {
						variant: 'World_Imagery',
						attribution:
							'{attribution.Esri} &mdash; ' +
							'Source: Esri, i-cubed, USDA, USGS, AEX, GeoEye, Getmapping, Aerogrid, IGN, IGP, UPR-EGP, and the GIS User Community'
					}
				},
				WorldTerrain: {
					options: {
						variant: 'World_Terrain_Base',
						maxZoom: 13,
						attribution:
							'{attribution.Esri} &mdash; ' +
							'Source: USGS, Esri, TANA, DeLorme, and NPS'
					}
				},
				WorldShadedRelief: {
					options: {
						variant: 'World_Shaded_Relief',
						maxZoom: 13,
						attribution: '{attribution.Esri} &mdash; Source: Esri'
					}
				},
				WorldPhysical: {
					options: {
						variant: 'World_Physical_Map',
						maxZoom: 8,
						attribution: '{attribution.Esri} &mdash; Source: US National Park Service'
					}
				},
				OceanBasemap: {
					options: {
						variant: 'Ocean/World_Ocean_Base',
						maxZoom: 13,
						attribution: '{attribution.Esri} &mdash; Sources: GEBCO, NOAA, CHS, OSU, UNH, CSUMB, National Geographic, DeLorme, NAVTEQ, and Esri'
					}
				},
				NatGeoWorldMap: {
					options: {
						variant: 'NatGeo_World_Map',
						maxZoom: 16,
						attribution: '{attribution.Esri} &mdash; National Geographic, Esri, DeLorme, NAVTEQ, UNEP-WCMC, USGS, NASA, ESA, METI, NRCAN, GEBCO, NOAA, iPC'
					}
				},
				WorldGrayCanvas: {
					options: {
						variant: 'Canvas/World_Light_Gray_Base',
						maxZoom: 16,
						attribution: '{attribution.Esri} &mdash; Esri, DeLorme, NAVTEQ'
					}
				}
			}
		},
		OpenWeatherMap: {
			url: 'http://{s}.tile.openweathermap.org/map/{variant}/{z}/{x}/{y}.png?appid={apiKey}',
			options: {
				maxZoom: 19,
				attribution: 'Map data &copy; <a href="http://openweathermap.org">OpenWeatherMap</a>',
				apiKey: '<insert your api key here>',
				opacity: 0.5
			},
			variants: {
				Clouds: 'clouds',
				CloudsClassic: 'clouds_cls',
				Precipitation: 'precipitation',
				PrecipitationClassic: 'precipitation_cls',
				Rain: 'rain',
				RainClassic: 'rain_cls',
				Pressure: 'pressure',
				PressureContour: 'pressure_cntr',
				Wind: 'wind',
				Temperature: 'temp',
				Snow: 'snow'
			}
		},
		HERE: {
			/*
			 * HERE maps, formerly Nokia maps.
			 * These basemaps are free, but you need an api id and app key. Please sign up at
			 * https://developer.here.com/plans
			 */
			url:
				'https://{s}.{base}.maps.api.here.com/maptile/2.1/' +
				'{type}/{mapID}/{variant}/{z}/{x}/{y}/{size}/{format}?' +
				'app_id={app_id}&app_code={app_code}&lg={language}',
			options: {
				attribution:
					'Map &copy; 1987-' + new Date().getFullYear() + ' <a href="http://developer.here.com">HERE</a>',
				subdomains: '1234',
				mapID: 'newest',
				'app_id': '<insert your app_id here>',
				'app_code': '<insert your app_code here>',
				base: 'base',
				variant: 'normal.day',
				maxZoom: 20,
				type: 'maptile',
				language: 'eng',
				format: 'png8',
				size: '256'
			},
			variants: {
				normalDay: 'normal.day',
				normalDayCustom: 'normal.day.custom',
				normalDayGrey: 'normal.day.grey',
				normalDayMobile: 'normal.day.mobile',
				normalDayGreyMobile: 'normal.day.grey.mobile',
				normalDayTransit: 'normal.day.transit',
				normalDayTransitMobile: 'normal.day.transit.mobile',
				normalDayTraffic: {
					options: {
						variant: 'normal.traffic.day',
						base: 'traffic',
						type: 'traffictile'
					}
				},
				normalNight: 'normal.night',
				normalNightMobile: 'normal.night.mobile',
				normalNightGrey: 'normal.night.grey',
				normalNightGreyMobile: 'normal.night.grey.mobile',
				normalNightTransit: 'normal.night.transit',
				normalNightTransitMobile: 'normal.night.transit.mobile',
				reducedDay: 'reduced.day',
				reducedNight: 'reduced.night',
				basicMap: {
					options: {
						type: 'basetile'
					}
				},
				mapLabels: {
					options: {
						type: 'labeltile',
						format: 'png'
					}
				},
				trafficFlow: {
					options: {
						base: 'traffic',
						type: 'flowtile'
					}
				},
				carnavDayGrey: 'carnav.day.grey',
				hybridDay: {
					options: {
						base: 'aerial',
						variant: 'hybrid.day'
					}
				},
				hybridDayMobile: {
					options: {
						base: 'aerial',
						variant: 'hybrid.day.mobile'
					}
				},
				hybridDayTransit: {
					options: {
						base: 'aerial',
						variant: 'hybrid.day.transit'
					}
				},
				hybridDayGrey: {
					options: {
						base: 'aerial',
						variant: 'hybrid.grey.day'
					}
				},
				hybridDayTraffic: {
					options: {
						variant: 'hybrid.traffic.day',
						base: 'traffic',
						type: 'traffictile'
					}
				},
				pedestrianDay: 'pedestrian.day',
				pedestrianNight: 'pedestrian.night',
				satelliteDay: {
					options: {
						base: 'aerial',
						variant: 'satellite.day'
					}
				},
				terrainDay: {
					options: {
						base: 'aerial',
						variant: 'terrain.day'
					}
				},
				terrainDayMobile: {
					options: {
						base: 'aerial',
						variant: 'terrain.day.mobile'
					}
				}
			}
		},
		HEREv3: {
			/*
			 * HERE maps API Version 3.
			 * These basemaps are free, but you need an API key. Please sign up at
			 * https://developer.here.com/plans
			 * Version 3 deprecates the app_id and app_code access in favor of apiKey
			 *
			 * Supported access methods as of 2019/12/21:
			 * @see https://developer.here.com/faqs#access-control-1--how-do-you-control-access-to-here-location-services
			 */
			url:
				'https://{s}.{base}.maps.ls.hereapi.com/maptile/2.1/' +
				'{type}/{mapID}/{variant}/{z}/{x}/{y}/{size}/{format}?' +
				'apiKey={apiKey}&lg={language}',
			options: {
				attribution:
					'Map &copy; 1987-' + new Date().getFullYear() + ' <a href="http://developer.here.com">HERE</a>',
				subdomains: '1234',
				mapID: 'newest',
				apiKey: '<insert your apiKey here>',
				base: 'base',
				variant: 'normal.day',
				maxZoom: 20,
				type: 'maptile',
				language: 'eng',
				format: 'png8',
				size: '256'
			},
			variants: {
				normalDay: 'normal.day',
				normalDayCustom: 'normal.day.custom',
				normalDayGrey: 'normal.day.grey',
				normalDayMobile: 'normal.day.mobile',
				normalDayGreyMobile: 'normal.day.grey.mobile',
				normalDayTransit: 'normal.day.transit',
				normalDayTransitMobile: 'normal.day.transit.mobile',
				normalNight: 'normal.night',
				normalNightMobile: 'normal.night.mobile',
				normalNightGrey: 'normal.night.grey',
				normalNightGreyMobile: 'normal.night.grey.mobile',
				normalNightTransit: 'normal.night.transit',
				normalNightTransitMobile: 'normal.night.transit.mobile',
				reducedDay: 'reduced.day',
				reducedNight: 'reduced.night',
				basicMap: {
					options: {
						type: 'basetile'
					}
				},
				mapLabels: {
					options: {
						type: 'labeltile',
						format: 'png'
					}
				},
				trafficFlow: {
					options: {
						base: 'traffic',
						type: 'flowtile'
					}
				},
				carnavDayGrey: 'carnav.day.grey',
				hybridDay: {
					options: {
						base: 'aerial',
						variant: 'hybrid.day'
					}
				},
				hybridDayMobile: {
					options: {
						base: 'aerial',
						variant: 'hybrid.day.mobile'
					}
				},
				hybridDayTransit: {
					options: {
						base: 'aerial',
						variant: 'hybrid.day.transit'
					}
				},
				hybridDayGrey: {
					options: {
						base: 'aerial',
						variant: 'hybrid.grey.day'
					}
				},
				pedestrianDay: 'pedestrian.day',
				pedestrianNight: 'pedestrian.night',
				satelliteDay: {
					options: {
						base: 'aerial',
						variant: 'satellite.day'
					}
				},
				terrainDay: {
					options: {
						base: 'aerial',
						variant: 'terrain.day'
					}
				},
				terrainDayMobile: {
					options: {
						base: 'aerial',
						variant: 'terrain.day.mobile'
					}
				}
			}
		},
		FreeMapSK: {
			url: 'https://{s}.freemap.sk/T/{z}/{x}/{y}.jpeg',
			options: {
				minZoom: 8,
				maxZoom: 16,
				subdomains: 'abcd',
				bounds: [[47.204642, 15.996093], [49.830896, 22.576904]],
				attribution:
					'{attribution.OpenStreetMap}, visualization CC-By-SA 2.0 <a href="http://freemap.sk">Freemap.sk</a>'
			}
		},
		MtbMap: {
			url: 'http://tile.mtbmap.cz/mtbmap_tiles/{z}/{x}/{y}.png',
			options: {
				attribution:
					'{attribution.OpenStreetMap} &amp; USGS'
			}
		},
		CartoDB: {
			url: 'https://{s}.basemaps.cartocdn.com/{variant}/{z}/{x}/{y}{r}.png',
			options: {
				attribution: '{attribution.OpenStreetMap} &copy; <a href="https://carto.com/attributions">CARTO</a>',
				subdomains: 'abcd',
				maxZoom: 20,
				variant: 'light_all'
			},
			variants: {
				Positron: 'light_all',
				PositronNoLabels: 'light_nolabels',
				PositronOnlyLabels: 'light_only_labels',
				DarkMatter: 'dark_all',
				DarkMatterNoLabels: 'dark_nolabels',
				DarkMatterOnlyLabels: 'dark_only_labels',
				Voyager: 'rastertiles/voyager',
				VoyagerNoLabels: 'rastertiles/voyager_nolabels',
				VoyagerOnlyLabels: 'rastertiles/voyager_only_labels',
				VoyagerLabelsUnder: 'rastertiles/voyager_labels_under'
			}
		},
		HikeBike: {
			url: 'https://tiles.wmflabs.org/{variant}/{z}/{x}/{y}.png',
			options: {
				maxZoom: 19,
				attribution: '{attribution.OpenStreetMap}',
				variant: 'hikebike'
			},
			variants: {
				HikeBike: {},
				HillShading: {
					options: {
						maxZoom: 15,
						variant: 'hillshading'
					}
				}
			}
		},
		BasemapAT: {
			url: 'https://mapsneu.wien.gv.at/basemap/{variant}/{type}/google3857/{z}/{y}/{x}.{format}',
			options: {
				maxZoom: 19,
				attribution: 'Datenquelle: <a href="https://www.basemap.at">basemap.at</a>',
				type: 'normal',
				format: 'png',
				bounds: [[46.358770, 8.782379], [49.037872, 17.189532]],
				variant: 'geolandbasemap'
			},
			variants: {
				basemap: {
					options: {
						maxZoom: 20, // currently only in Vienna
						variant: 'geolandbasemap'
					}
				},
				grau: 'bmapgrau',
				overlay: 'bmapoverlay',
				terrain: {
					options: {
						variant: 'bmapgelaende',
						type: 'grau',
						format: 'jpeg'
					}
				},
				surface: {
					options: {
						variant: 'bmapoberflaeche',
						type: 'grau',
						format: 'jpeg'
					}
				},
				highdpi: {
					options: {
						variant: 'bmaphidpi',
						format: 'jpeg'
					}
				},
				orthofoto: {
					options: {
						maxZoom: 20, // currently only in Vienna
						variant: 'bmaporthofoto30cm',
						format: 'jpeg'
					}
				}
			}
		},
		nlmaps: {
			url: 'https://service.pdok.nl/brt/achtergrondkaart/wmts/v2_0/{variant}/EPSG:3857/{z}/{x}/{y}.png',
			options: {
				minZoom: 6,
				maxZoom: 19,
				bounds: [[50.5, 3.25], [54, 7.6]],
				attribution: 'Kaartgegevens &copy; <a href="https://www.kadaster.nl">Kadaster</a>'
			},
			variants: {
				'standaard': 'standaard',
				'pastel': 'pastel',
				'grijs': 'grijs',
				'water': 'water',
				'luchtfoto': {
					'url': 'https://service.pdok.nl/hwh/luchtfotorgb/wmts/v1_0/Actueel_ortho25/EPSG:3857/{z}/{x}/{y}.jpeg',
				}
			}
		},
		NASAGIBS: {
			url: 'https://map1.vis.earthdata.nasa.gov/wmts-webmerc/{variant}/default/{time}/{tilematrixset}{maxZoom}/{z}/{y}/{x}.{format}',
			options: {
				attribution:
					'Imagery provided by services from the Global Imagery Browse Services (GIBS), operated by the NASA/GSFC/Earth Science Data and Information System ' +
					'(<a href="https://earthdata.nasa.gov">ESDIS</a>) with funding provided by NASA/HQ.',
				bounds: [[-85.0511287776, -179.999999975], [85.0511287776, 179.999999975]],
				minZoom: 1,
				maxZoom: 9,
				format: 'jpg',
				time: '',
				tilematrixset: 'GoogleMapsCompatible_Level'
			},
			variants: {
				ModisTerraTrueColorCR: 'MODIS_Terra_CorrectedReflectance_TrueColor',
				ModisTerraBands367CR: 'MODIS_Terra_CorrectedReflectance_Bands367',
				ViirsEarthAtNight2012: {
					options: {
						variant: 'VIIRS_CityLights_2012',
						maxZoom: 8
					}
				},
				ModisTerraLSTDay: {
					options: {
						variant: 'MODIS_Terra_Land_Surface_Temp_Day',
						format: 'png',
						maxZoom: 7,
						opacity: 0.75
					}
				},
				ModisTerraSnowCover: {
					options: {
						variant: 'MODIS_Terra_NDSI_Snow_Cover',
						format: 'png',
						maxZoom: 8,
						opacity: 0.75
					}
				},
				ModisTerraAOD: {
					options: {
						variant: 'MODIS_Terra_Aerosol',
						format: 'png',
						maxZoom: 6,
						opacity: 0.75
					}
				},
				ModisTerraChlorophyll: {
					options: {
						variant: 'MODIS_Terra_Chlorophyll_A',
						format: 'png',
						maxZoom: 7,
						opacity: 0.75
					}
				}
			}
		},
		NLS: {
			// NLS maps are copyright National library of Scotland.
			// http://maps.nls.uk/projects/api/index.html
			// Please contact NLS for anything other than non-commercial low volume usage
			//
			// Map sources: Ordnance Survey 1:1m to 1:63K, 1920s-1940s
			//   z0-9  - 1:1m
			//  z10-11 - quarter inch (1:253440)
			//  z12-18 - one inch (1:63360)
			url: 'https://nls-{s}.tileserver.com/nls/{z}/{x}/{y}.jpg',
			options: {
				attribution: '<a href="http://geo.nls.uk/maps/">National Library of Scotland Historic Maps</a>',
				bounds: [[49.6, -12], [61.7, 3]],
				minZoom: 1,
				maxZoom: 18,
				subdomains: '0123',
			}
		},
		JusticeMap: {
			// Justice Map (http://www.justicemap.org/)
			// Visualize race and income data for your community, county and country.
			// Includes tools for data journalists, bloggers and community activists.
			url: 'https://www.justicemap.org/tile/{size}/{variant}/{z}/{x}/{y}.png',
			options: {
				attribution: '<a href="http://www.justicemap.org/terms.php">Justice Map</a>',
				// one of 'county', 'tract', 'block'
				size: 'county',
				// Bounds for USA, including Alaska and Hawaii
				bounds: [[14, -180], [72, -56]]
			},
			variants: {
				income: 'income',
				americanIndian: 'indian',
				asian: 'asian',
				black: 'black',
				hispanic: 'hispanic',
				multi: 'multi',
				nonWhite: 'nonwhite',
				white: 'white',
				plurality: 'plural'
			}
		},
		GeoportailFrance: {
			url: 'https://wxs.ign.fr/{apikey}/geoportail/wmts?REQUEST=GetTile&SERVICE=WMTS&VERSION=1.0.0&STYLE={style}&TILEMATRIXSET=PM&FORMAT={format}&LAYER={variant}&TILEMATRIX={z}&TILEROW={y}&TILECOL={x}',
			options: {
				attribution: '<a target="_blank" href="https://www.geoportail.gouv.fr/">Geoportail France</a>',
				bounds: [[-75, -180], [81, 180]],
				minZoom: 2,
				maxZoom: 18,
				// Get your own geoportail apikey here : http://professionnels.ign.fr/ign/contrats/
				// NB : 'choisirgeoportail' is a demonstration key that comes with no guarantee
				apikey: 'choisirgeoportail',
				format: 'image/png',
				style: 'normal',
				variant: 'GEOGRAPHICALGRIDSYSTEMS.PLANIGNV2'
			},
			variants: {
				plan: 'GEOGRAPHICALGRIDSYSTEMS.PLANIGNV2',
				parcels: {
					options: {
						variant: 'CADASTRALPARCELS.PARCELLAIRE_EXPRESS',
						style: 'PCI vecteur',
						maxZoom: 20
					}
				},
				orthos: {
					options: {
						maxZoom: 19,
						format: 'image/jpeg',
						variant: 'ORTHOIMAGERY.ORTHOPHOTOS'
					}
				}
			}
		},
		OneMapSG: {
			url: 'https://maps-{s}.onemap.sg/v3/{variant}/{z}/{x}/{y}.png',
			options: {
				variant: 'Default',
				minZoom: 11,
				maxZoom: 18,
				bounds: [[1.56073, 104.11475], [1.16, 103.502]],
				attribution: '<img src="https://docs.onemap.sg/maps/images/oneMap64-01.png" style="height:20px;width:20px;"/> New OneMap | Map data &copy; contributors, <a href="http://SLA.gov.sg">Singapore Land Authority</a>'
			},
			variants: {
				Default: 'Default',
				Night: 'Night',
				Original: 'Original',
				Grey: 'Grey',
				LandLot: 'LandLot'
			}
		},
		USGS: {
			url: 'https://basemap.nationalmap.gov/arcgis/rest/services/USGSTopo/MapServer/tile/{z}/{y}/{x}',
			options: {
				maxZoom: 20,
				attribution: 'Tiles courtesy of the <a href="https://usgs.gov/">U.S. Geological Survey</a>'
			},
			variants: {
				USTopo: {},
				USImagery: {
					url: 'https://basemap.nationalmap.gov/arcgis/rest/services/USGSImageryOnly/MapServer/tile/{z}/{y}/{x}'
				},
				USImageryTopo: {
					url: 'https://basemap.nationalmap.gov/arcgis/rest/services/USGSImageryTopo/MapServer/tile/{z}/{y}/{x}'
				}
			}
		},
		WaymarkedTrails: {
			url: 'https://tile.waymarkedtrails.org/{variant}/{z}/{x}/{y}.png',
			options: {
				maxZoom: 18,
				attribution: 'Map data: {attribution.OpenStreetMap} | Map style: &copy; <a href="https://waymarkedtrails.org">waymarkedtrails.org</a> (<a href="https://creativecommons.org/licenses/by-sa/3.0/">CC-BY-SA</a>)'
			},
			variants: {
				hiking: 'hiking',
				cycling: 'cycling',
				mtb: 'mtb',
				slopes: 'slopes',
				riding: 'riding',
				skating: 'skating'
			}
		},
		OpenAIP: {
			url: 'https://{s}.tile.maps.openaip.net/geowebcache/service/tms/1.0.0/openaip_basemap@EPSG%3A900913@png/{z}/{x}/{y}.{ext}',
			options: {
				attribution: '<a href="https://www.openaip.net/">openAIP Data</a> (<a href="https://creativecommons.org/licenses/by-sa/3.0/">CC-BY-NC-SA</a>)',
				ext: 'png',
				minZoom: 4,
				maxZoom: 14,
				tms: true,
				detectRetina: true,
				subdomains: '12'
			}
		},
		OpenSnowMap: {
			url: 'https://tiles.opensnowmap.org/{variant}/{z}/{x}/{y}.png',
			options: {
				minZoom: 9,
				maxZoom: 18,
				attribution: 'Map data: {attribution.OpenStreetMap} & ODbL, &copy; <a href="https://www.opensnowmap.org/iframes/data.html">www.opensnowmap.org</a> <a href="https://creativecommons.org/licenses/by-sa/2.0/">CC-BY-SA</a>'
			},
			variants: {
				pistes: 'pistes',
			}
		},
		AzureMaps: {
			url: 
				'https://atlas.microsoft.com/map/tile?api-version={apiVersion}'+
				'&tilesetId={variant}&x={x}&y={y}&zoom={z}&language={language}'+
				'&subscription-key={subscriptionKey}',
			options: {
				attribution: 'See https://docs.microsoft.com/en-us/rest/api/maps/render-v2/get-map-tile for details.',
				apiVersion: '2.0',
				variant: 'microsoft.imagery',
				subscriptionKey: '<insert your subscription key here>',
				language: 'en-US',
			},
			variants: {
				MicrosoftImagery: 'microsoft.imagery',
				MicrosoftBaseDarkGrey: 'microsoft.base.darkgrey',
				MicrosoftBaseRoad: 'microsoft.base.road',
				MicrosoftBaseHybridRoad: 'microsoft.base.hybrid.road',
				MicrosoftTerraMain: 'microsoft.terra.main',
				MicrosoftWeatherInfraredMain: {
					url: 
					'https://atlas.microsoft.com/map/tile?api-version={apiVersion}'+
					'&tilesetId={variant}&x={x}&y={y}&zoom={z}'+
					'&timeStamp={timeStamp}&language={language}' +
					'&subscription-key={subscriptionKey}',
					options: {
						timeStamp: '2021-05-08T09:03:00Z',
						attribution: 'See https://docs.microsoft.com/en-us/rest/api/maps/render-v2/get-map-tile#uri-parameters for details.',
						variant: 'microsoft.weather.infrared.main',
					},
				},
				MicrosoftWeatherRadarMain: {
					url: 
					'https://atlas.microsoft.com/map/tile?api-version={apiVersion}'+
					'&tilesetId={variant}&x={x}&y={y}&zoom={z}'+
					'&timeStamp={timeStamp}&language={language}' +
					'&subscription-key={subscriptionKey}',
					options: {
						timeStamp: '2021-05-08T09:03:00Z',
						attribution: 'See https://docs.microsoft.com/en-us/rest/api/maps/render-v2/get-map-tile#uri-parameters for details.',
						variant: 'microsoft.weather.radar.main',
					},
				}
			},
		},
		SwissFederalGeoportal: {
			url: 'https://wmts.geo.admin.ch/1.0.0/{variant}/default/current/3857/{z}/{x}/{y}.jpeg',
			options: {
				attribution: '&copy; <a href="https://www.swisstopo.admin.ch/">swisstopo</a>',
				minZoom: 2,
				maxZoom: 18,
				bounds: [[45.398181, 5.140242], [48.230651, 11.47757]]
			},
			variants: {
				NationalMapColor: 'ch.swisstopo.pixelkarte-farbe',
				NationalMapGrey: 'ch.swisstopo.pixelkarte-grau',
				SWISSIMAGE: {
					options: {
						variant: 'ch.swisstopo.swissimage',
						maxZoom: 19
					}
				}
			}
		}
	};

	L.tileLayer.provider = function (provider, options) {
		return new L.TileLayer.Provider(provider, options);
	};

	return L;
}));
</script>
<script>LeafletWidget.methods.addProviderTiles = function(provider, layerId, group, options) {
  this.layerManager.addLayer(L.tileLayer.provider(provider, options), "tile", layerId, group);
};
</script>

</head>
<body style="background-color: white;">
<div id="htmlwidget_container">
  <div class="leaflet html-widget html-fill-item-overflow-hidden html-fill-item" id="htmlwidget-8f7a3f3bb4e8c93e365c" style="width:100%;height:400px;"></div>
</div>
<script type="application/json" data-for="htmlwidget-8f7a3f3bb4e8c93e365c">{"x":{"options":{"crs":{"crsClass":"L.CRS.EPSG3857","code":null,"proj4def":null,"projectedBounds":null,"options":{}}},"calls":[{"method":"addProviderTiles","args":["Esri.WorldImagery",null,"Esri.WorldImagery",{"errorTileUrl":"","noWrap":false,"detectRetina":false}]},{"method":"addPolygons","args":[[[[{"lng":[-27.9437504217017,-27.94402853,-27.94402853,-27.9437504217017],"lat":[39.0491797957035,39.049130608248,39.0505053820862,39.0491797957035]}]],[[{"lng":[-27.953449636,-27.953705904,-27.952152116,-27.951798064,-27.953449636],"lat":[39.052897749,39.054413692,39.054413094,39.052997657,39.052897749]}]],[[{"lng":[-27.956717428,-27.9571961160716,-27.9532413466074,-27.9531966525416,-27.956717428],"lat":[39.046886394,39.0475895794803,39.047501189027,39.0475090938158,39.046886394]}],[{"lng":[-27.9650544802435,-27.948940124,-27.9474815645336,-27.9531465681917,-27.95455641,-27.957555554,-27.9649869212672,-27.9650544802435],"lat":[39.0591333966603,39.061356858,39.0485198893898,39.047517951954,39.053899264,39.059323849,39.059034153554,39.0591333966603]},{"lng":[-27.953705904,-27.953449636,-27.951798064,-27.952152116,-27.953705904],"lat":[39.054413692,39.052897749,39.052997657,39.054413094,39.054413692]}]],[[{"lng":[-27.9474815645336,-27.9470816982896,-27.9595376070619,-27.9684082819491,-27.9571961160716,-27.956717428,-27.9532413466074,-27.953142376,-27.9531465681917,-27.9474815645336],"lat":[39.0485198893898,39.0450006153799,39.0460180980249,39.0478401752313,39.0475895794803,39.046886394,39.047501189027,39.047498977,39.047517951954,39.0485198893898]}],[{"lng":[-27.9703445890387,-27.9702909860752,-27.9684637374809,-27.9696737397013,-27.9703445890387],"lat":[39.0495538448187,39.0497625586229,39.0478515660512,39.0481001060433,39.0495538448187]}],[{"lng":[-27.9660835656958,-27.9650544802435,-27.9649869212672,-27.9660835656958],"lat":[39.0589914032921,39.0591333966603,39.059034153554,39.0589914032921]}]],[[{"lng":[-27.9532413466074,-27.9531465681917,-27.953142376,-27.9532413466074],"lat":[39.047501189027,39.047517951954,39.047498977,39.047501189027]}],[{"lng":[-27.9702909860752,-27.969481258,-27.969992659,-27.9660835656958,-27.9649869212672,-27.9571961160716,-27.9684082819491,-27.9684637374809,-27.9702909860752],"lat":[39.0497625586229,39.052915396,39.058452026,39.0589914032921,39.059034153554,39.0475895794803,39.0478401752313,39.0478515660512,39.0497625586229]}]],[[{"lng":[-27.9531966525416,-27.9532413466074,-27.9571961160716,-27.9649869212672,-27.957555554,-27.95455641,-27.9531465681917,-27.9531966525416],"lat":[39.0475090938158,39.047501189027,39.0475895794803,39.059034153554,39.059323849,39.053899264,39.047517951954,39.0475090938158]}]],[[{"lng":[-27.9710489712481,-27.9705807447239,-27.971018195,-27.9710489712481],"lat":[39.0510802497407,39.0500655970966,39.050523096,39.0510802497407]}]],[[{"lng":[-27.9684082819491,-27.968453819,-27.9684637374809,-27.9684434419274,-27.9684082819491],"lat":[39.0478401752313,39.047841193,39.0478515660512,39.0478473972517,39.0478401752313]}],[{"lng":[-27.9710489712481,-27.971474362,-27.9660835656958,-27.969992659,-27.969481258,-27.9702909860752,-27.9705807447239,-27.9706937223212,-27.9710489712481],"lat":[39.0510802497407,39.058781255,39.0589914032921,39.058452026,39.052915396,39.0497625586229,39.0500655970966,39.0503104209413,39.0510802497407]}]],[[{"lng":[-27.940502552,-27.924681145,-27.922797645,-27.947659855,-27.952185908,-27.940502552],"lat":[39.017329408,39.017036729,38.999180964,38.996546125,39.008260502,39.017329408]}]],[[{"lng":[-27.94402853,-27.94402853,-27.946755293,-27.9638603065057,-27.9614689405472,-27.94402853],"lat":[39.065565165,39.0505053820862,39.063502333,39.0633864982964,39.0646442867905,39.065565165]}]],[[{"lng":[-27.9437504217017,-27.94402853,-27.94402853,-27.9614689405472,-27.955193859,-27.940502552,-27.93522875,-27.9437504217017],"lat":[39.0491797957035,39.0505053820862,39.065565165,39.0646442867905,39.067944796,39.067067381,39.050686976,39.0491797957035]}]],[[{"lng":[-27.976931536,-27.9614689405472,-27.9638603065057,-27.9766877860887,-27.976931536],"lat":[39.06382784,39.0646442867905,39.0633864982964,39.0632996309405,39.06382784]}]],[[{"lng":[-27.946902421,-27.9475881094037,-27.9469262802109,-27.946902421],"lat":[39.043422773,39.0435636165352,39.0436327609697,39.043422773]}]],[[{"lng":[-27.9470816982896,-27.94402853,-27.94402853,-27.9437504217017,-27.942679732,-27.9469262802109,-27.9470816982896],"lat":[39.0450006153799,39.044751212,39.049130608248,39.0491797957035,39.044076418,39.0436327609697,39.0450006153799]}],[{"lng":[-27.9475881094037,-27.974792441,-27.975328637,-27.978102579,-27.9766877860887,-27.9710489712481,-27.971018195,-27.9705807447239,-27.9703445890387,-27.970665621,-27.9696737397013,-27.969072385,-27.9595376070619,-27.9475881094037],"lat":[39.0435636165352,39.040721451,39.04120533,39.06329005,39.0632996309405,39.0510802497407,39.050523096,39.0500655970966,39.0495538448187,39.048303843,39.0481001060433,39.046796963,39.0460180980249,39.0435636165352]}],[{"lng":[-27.9638603065057,-27.9638603065057,-27.966871564,-27.9638603065057],"lat":[39.0633864982964,39.0633864982964,39.061802665,39.0633864982964]}]],[[{"lng":[-27.9469262802109,-27.9475881094037,-27.9595376070619,-27.9470816982896,-27.9469262802109],"lat":[39.0436327609697,39.0435636165352,39.0460180980249,39.0450006153799,39.0436327609697]}],[{"lng":[-27.970665621,-27.9703445890387,-27.9696737397013,-27.970665621],"lat":[39.048303843,39.0495538448187,39.0481001060433,39.048303843]}]],[[{"lng":[-27.9684082819491,-27.9684434419274,-27.9684082819491,-27.9684082819491],"lat":[39.0478401752313,39.0478473972517,39.0478401752313,39.0478401752313]},{"lng":[-27.9571961160716,-27.9649869212672,-27.9571961160716,-27.9571961160716],"lat":[39.0475895794803,39.059034153554,39.0475895794803,39.0475895794803]}]],[[{"lng":[-27.9474815645336,-27.94402853,-27.94402853,-27.9470816982896,-27.9474815645336],"lat":[39.0485198893898,39.049130608248,39.044751212,39.0450006153799,39.0485198893898]}],[{"lng":[-27.969072385,-27.9696737397013,-27.9684434419274,-27.9684637374809,-27.968453819,-27.9684082819491,-27.9595376070619,-27.969072385],"lat":[39.046796963,39.0481001060433,39.0478473972517,39.0478515660512,39.047841193,39.0478401752313,39.0460180980249,39.046796963]}],[{"lng":[-27.9705807447239,-27.9702909860752,-27.969481258,-27.9703445890387,-27.9705807447239],"lat":[39.0500655970966,39.0497625586229,39.052915396,39.0495538448187,39.0500655970966]}],[{"lng":[-27.9710489712481,-27.9706937223212,-27.9766877860887,-27.9638603065057,-27.966871564,-27.9650544802435,-27.969992659,-27.9660835656958,-27.971474362,-27.9710489712481],"lat":[39.0510802497407,39.0503104209413,39.0632996309405,39.0633864982964,39.061802665,39.0591333966603,39.058452026,39.0589914032921,39.058781255,39.0510802497407]}]],[[{"lng":[-27.9638603065057,-27.946755293,-27.94402853,-27.94402853,-27.9474815645336,-27.948940124,-27.9650544802435,-27.966871564,-27.9638603065057],"lat":[39.0633864982964,39.063502333,39.0505053820862,39.049130608248,39.0485198893898,39.061356858,39.0591333966603,39.061802665,39.0633864982964]}]]],null,"Conflicts overlay",{"interactive":true,"className":"","stroke":true,"color":"black","weight":0.2,"opacity":0.5,"fill":true,"fillColor":["#FCAE91","#A50F15","#DE2D26","#FB6A4A","#DE2D26","#A50F15","#FCAE91","#FB6A4A","#FEE5D9","#FCAE91","#FEE5D9","#FEE5D9","#FEE5D9","#FEE5D9","#FCAE91","#FEE5D9","#FCAE91","#FB6A4A"],"fillOpacity":0.8,"smoothFactor":0.2,"noClip":false},["<br>Name Area: Zona de conflitos,Ilhéu da Praia <br>Institution: Clube Naval Graciosa,SPEA <br>Superficie (km²): 0.002","<br>Name Area: Zona de conflitos,Ilhéu da Praia e áreas adjacentes,Location 1,Ilhéu da Praia,Sra da Guia <br>Institution: Clube Naval Graciosa,Secretaria Regional do Ambiente e Altera,Associacao de Pescadores Graciosenses,SPEA,DivinGraciosa <br>Superficie (km²): 0.023","<br>Name Area: Zona de conflitos,Ilhéu da Praia e áreas adjacentes,Location 1,Ilhéu da Praia <br>Institution: Clube Naval Graciosa,Secretaria Regional do Ambiente e Altera,Associacao de Pescadores Graciosenses,SPEA <br>Superficie (km²): 0.892","<br>Name Area: Zona de conflitos,Ilhéu da Praia e áreas adjacentes,Location 1 <br>Institution: Clube Naval Graciosa,Secretaria Regional do Ambiente e Altera,Associacao de Pescadores Graciosenses <br>Superficie (km²): 0.335","<br>Name Area: Location 1,Zona de conflitos,Ilhéu da Praia e áreas adjacentes,Location 1 <br>Institution: DRPM,Clube Naval Graciosa,Secretaria Regional do Ambiente e Altera,Associacao de Pescadores Graciosenses <br>Superficie (km²): 0.921","<br>Name Area: Location 1,Zona de conflitos,Ilhéu da Praia e áreas adjacentes,Location 1,Ilhéu da Praia <br>Institution: DRPM,Clube Naval Graciosa,Secretaria Regional do Ambiente e Altera,Associacao de Pescadores Graciosenses,SPEA <br>Superficie (km²): 0.703","<br>Name Area: Location 1,Zona de conflitos <br>Institution: DRPM,Clube Naval Graciosa <br>Superficie (km²): 0.001","<br>Name Area: Location 1,Zona de conflitos,Location 1 <br>Institution: DRPM,Clube Naval Graciosa,Associacao de Pescadores Graciosenses <br>Superficie (km²): 0.128","<br>Name Area: Ilhéu de Baixo <br>Institution: SPEA <br>Superficie (km²): 4.605","<br>Name Area: Location 1,Ilhéu da Praia <br>Institution: Associacao de Pescadores Graciosenses,SPEA <br>Superficie (km²): 0.46","<br>Name Area: Ilhéu da Praia <br>Institution: SPEA <br>Superficie (km²): 1.378","<br>Name Area: Location 1 <br>Institution: Associacao de Pescadores Graciosenses <br>Superficie (km²): 0.117","<br>Name Area: Ilhéu da Praia e áreas adjacentes <br>Institution: Secretaria Regional do Ambiente e Altera <br>Superficie (km²): 0.001","<br>Name Area: Zona de conflitos <br>Institution: Clube Naval Graciosa <br>Superficie (km²): 1.728","<br>Name Area: Zona de conflitos,Ilhéu da Praia e áreas adjacentes <br>Institution: Clube Naval Graciosa,Secretaria Regional do Ambiente e Altera <br>Superficie (km²): 0.099","<br>Name Area: Location 1 <br>Institution: DRPM <br>Superficie (km²): 0","<br>Name Area: Zona de conflitos,Location 1 <br>Institution: Clube Naval Graciosa,Associacao de Pescadores Graciosenses <br>Superficie (km²): 0.725","<br>Name Area: Zona de conflitos,Location 1,Ilhéu da Praia <br>Institution: Clube Naval Graciosa,Associacao de Pescadores Graciosenses,SPEA <br>Superficie (km²): 0.958"],null,null,{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},{"color":"lightblue","weight":6}]},{"method":"addPolygons","args":[[[[{"lng":[-27.978102579,-27.946755293,-27.942679732,-27.974792441,-27.975328637,-27.978102579],"lat":[39.06329005,39.063502333,39.044076418,39.040721451,39.04120533,39.06329005]}]],[[{"lng":[-27.976931536,-27.94402853,-27.94402853,-27.969072385,-27.976931536],"lat":[39.06382784,39.065565165,39.044751212,39.046796963,39.06382784]}]],[[{"lng":[-27.940502552,-27.924681145,-27.922797645,-27.947659855,-27.952185908,-27.940502552],"lat":[39.017329408,39.017036729,38.999180964,38.996546125,39.008260502,39.017329408]}]],[[{"lng":[-27.955193859,-27.940502552,-27.93522875,-27.956717428,-27.966871564,-27.955193859],"lat":[39.067944796,39.067067381,39.050686976,39.046886394,39.061802665,39.067944796]}]],[[{"lng":[-27.969992659,-27.948940124,-27.946902421,-27.970665621,-27.969481258,-27.969992659],"lat":[39.058452026,39.061356858,39.043422773,39.048303843,39.052915396,39.058452026]}]],[[{"lng":[-27.971474362,-27.957555554,-27.95455641,-27.953142376,-27.968453819,-27.971018195,-27.971474362],"lat":[39.058781255,39.059323849,39.053899264,39.047498977,39.047841193,39.050523096,39.058781255]}]],[[{"lng":[-27.953705904,-27.952152116,-27.951798064,-27.953449636,-27.953705904],"lat":[39.054413692,39.054413094,39.052997657,39.052897749,39.054413692]}]]],null,"Conflicts",{"interactive":true,"className":"","stroke":true,"color":"black","weight":1,"opacity":0.5,"fill":true,"fillColor":"black","fillOpacity":0,"smoothFactor":0.2,"noClip":false},["<br>Name Area: Zona de conflitos <br>Institution: Clube Naval Graciosa <br>Description: pesca e conservação, falta de informação cientifica para que a proibição seja consistente, <br>Superficie (km²): 6.516195","<br>Name Area: Location 1 <br>Institution: Associacao de Pescadores Graciosenses <br>Description: Conflitos entre as entidades de conservação, pescas, marítima de recreio e mergulho <br>Superficie (km²): 5.262596","<br>Name Area: Ilhéu de Baixo <br>Institution: SPEA <br>Description: Pesca dentro da área da reserva, pouca fiscalização. <br>Superficie (km²): 4.604649","<br>Name Area: Ilhéu da Praia <br>Institution: SPEA <br>Description: Pesca dentro da área da reserva, pouca fiscalização. <br>Superficie (km²): 4.416803","<br>Name Area: Ilhéu da Praia e áreas adjacentes <br>Institution: Secretaria Regional do Ambiente e Altera <br>Description: Deve-se ao facto de ser uma zona estratégica para muitas atividades económicas (Pesca, Transporte, comércio), além disso é uma área fundamental de proteção ambiental <br>Superficie (km²): 2.974215","<br>Name Area: Location 1 <br>Institution: DRPM <br>Description: Conflitos entre a pesca, a navegação, o fundeio, a atividade portuária, o recreio, a visitação do património cultural subaquático, o mergulho <br>Superficie (km²): 1.753466","<br>Name Area: Sra da Guia <br>Institution: DivinGraciosa <br>Description: Zona utilizada pela pesca e mergulho <br>Superficie (km²): 0.022719"],null,null,{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},{"color":"lightblue","weight":6}]},{"method":"addLegend","args":[{"colors":["#FEE5D9","#FCAE91","#FB6A4A","#DE2D26","#A50F15"],"labels":["1","2","3","4","5"],"na_color":null,"na_label":"NA","opacity":0.5,"position":"topright","type":"factor","title":"Overlay Conflicts","extra":null,"layerId":null,"className":"info legend","group":null}]},{"method":"addPolygons","args":[[[[{"lng":[-27.93556192,-27.9413248373213,-27.94147399,-27.9416867277187,-27.941439912,-27.93556192],"lat":[39.045514743,39.0461150364807,39.049049607,39.0621828469047,39.062091384,39.045514743]}],[{"lng":[-27.9666480887182,-27.9680366129878,-27.969592403,-27.955052106,-27.9418802944526,-27.9485070015948,-27.948512821,-27.9666480887182],"lat":[39.061782676953,39.0617562214277,39.064733678,39.067135677,39.0622545772459,39.0621283187262,39.062675116,39.061782676953]}]],[[{"lng":[-27.9671661102714,-27.968936987,-27.9693877358163,-27.9671661102714],"lat":[39.0477312220434,39.04766018,39.0494739170023,39.0477312220434]}],[{"lng":[-27.970443263,-27.9706698618161,-27.9714320180988,-27.970443263],"lat":[39.057603858,39.0546329751815,39.0576997631607,39.057603858]}]],[[{"lng":[-27.9533331515843,-27.960491684,-27.9533331515843,-27.9533331515843],"lat":[39.052552283158,39.052327339,39.052552283158,39.052552283158]}]],[[{"lng":[-27.9597114797987,-27.9413248373213,-27.941153052,-27.961130808,-27.9671661102714,-27.9597114797987],"lat":[39.0480302785352,39.0461150364807,39.04273517,39.04299699,39.0477312220434,39.0480302785352]}],[{"lng":[-27.9706698618161,-27.9693877358163,-27.970968766,-27.9706698618161],"lat":[39.0546329751815,39.0494739170023,39.050714114,39.0546329751815]}],[{"lng":[-27.972376643,-27.9714320180988,-27.976404589,-27.978717528,-27.976661113,-27.9680366129878,-27.9680152813895,-27.972376643],"lat":[39.061500774,39.0576997631607,39.058182082,39.060167286,39.061591899,39.0617562214277,39.0617153972115,39.061500774]}],[{"lng":[-27.9418802944526,-27.941687949,-27.9416867277187,-27.9418802944526],"lat":[39.0622545772459,39.062258242,39.0621828469047,39.0622545772459]}]],[[{"lng":[-27.921640358,-27.920402886,-27.950720953,-27.950468809,-27.938347376,-27.921640358],"lat":[39.01307093,38.998406248,38.997684956,39.015748113,39.01571555,39.01307093]},{"lng":[-27.942600556,-27.938458681,-27.935973557,-27.940943806,-27.942600556],"lat":[39.009003727,39.005624293,39.007555418,39.010291088,39.009003727]}]],[[{"lng":[-27.9483969346723,-27.9485070015948,-27.9666480887182,-27.9680152813895,-27.948512821,-27.9483969346723],"lat":[39.0517863180137,39.0621283187262,39.061782676953,39.0617153972115,39.062675116,39.0517863180137]}]],[[{"lng":[-27.938458681,-27.942600556,-27.940943806,-27.935973557,-27.938458681],"lat":[39.005624293,39.009003727,39.010291088,39.007555418,39.005624293]}]],[[{"lng":[-27.9533331515843,-27.953162232,-27.9533331515843,-27.9532038407521,-27.9533331515843],"lat":[39.052552283158,39.052439219,39.052552283158,39.0525563465211,39.052552283158]}]],[[{"lng":[-27.951091294,-27.956268638,-27.958753762,-27.9576505620795,-27.9605263857036,-27.961980112,-27.951673081,-27.9506025151241,-27.9545572395054,-27.954818981,-27.951091294],"lat":[39.057906979,39.060158287,39.056138044,39.0554082717469,39.0525006146778,39.059759492,39.060656004,39.0566894213243,39.0563661277127,39.057102923,39.057906979]}]],[[{"lng":[-27.9533331515843,-27.9532038407521,-27.953162232,-27.9533331515843],"lat":[39.052552283158,39.0525563465211,39.052439219,39.052552283158]}]],[[{"lng":[-27.9533331515843,-27.953162232,-27.9532038407521,-27.949518271,-27.9506025151241,-27.94915764,-27.9483969346723,-27.9483804689418,-27.961632178,-27.9605263857036,-27.960491684,-27.9533331515843],"lat":[39.052552283158,39.052439219,39.0525563465211,39.052672159,39.0566894213243,39.056807538,39.0517863180137,39.0502391811191,39.051382582,39.0525006146778,39.052327339,39.052552283158]}]],[[{"lng":[-27.9533331515843,-27.9532038407521,-27.9533331515843,-27.9533331515843],"lat":[39.052552283158,39.0525563465211,39.052552283158,39.052552283158]}],[{"lng":[-27.960491684,-27.9605263857036,-27.9576505620795,-27.9533331515843,-27.960491684],"lat":[39.052327339,39.0525006146778,39.0554082717469,39.052552283158,39.052327339]}],[{"lng":[-27.949518271,-27.9532038407521,-27.9545572395054,-27.9506025151241,-27.949518271],"lat":[39.052672159,39.0525563465211,39.0563661277127,39.0566894213243,39.052672159]}]],[[{"lng":[-27.94147399,-27.9413248373213,-27.9597114797987,-27.948361806,-27.9483804689418,-27.94815966,-27.9483969346723,-27.9485070015948,-27.9418802944526,-27.9416867277187,-27.94147399],"lat":[39.049049607,39.0461150364807,39.0480302785352,39.048485592,39.0502391811191,39.050220129,39.0517863180137,39.0621283187262,39.0622545772459,39.0621828469047,39.049049607]}],[{"lng":[-27.9680366129878,-27.9666480887182,-27.9680152813895,-27.9680366129878],"lat":[39.0617562214277,39.061782676953,39.0617153972115,39.0617562214277]}]],[[{"lng":[-27.9706698618161,-27.970443263,-27.9714320180988,-27.972376643,-27.9680152813895,-27.960930859,-27.9597114797987,-27.9671661102714,-27.9693877358163,-27.9706698618161],"lat":[39.0546329751815,39.057603858,39.0576997631607,39.061500774,39.0617153972115,39.048157295,39.0480302785352,39.0477312220434,39.0494739170023,39.0546329751815]}]],[[{"lng":[-27.951091294,-27.954818981,-27.9545572395054,-27.956891964,-27.9576505620795,-27.958753762,-27.956268638,-27.951091294],"lat":[39.057906979,39.057102923,39.0563661277127,39.056175267,39.0554082717469,39.056138044,39.060158287,39.057906979]}]],[[{"lng":[-27.9532038407521,-27.9533331515843,-27.9576505620795,-27.956891964,-27.9545572395054,-27.9532038407521],"lat":[39.0525563465211,39.052552283158,39.0554082717469,39.056175267,39.0563661277127,39.0525563465211]}]],[[{"lng":[-27.9483969346723,-27.94915764,-27.9506025151241,-27.951673081,-27.961980112,-27.9605263857036,-27.961632178,-27.9483804689418,-27.948361806,-27.9597114797987,-27.960930859,-27.9680152813895,-27.9666480887182,-27.9485070015948,-27.9483969346723],"lat":[39.0517863180137,39.056807538,39.0566894213243,39.060656004,39.059759492,39.0525006146778,39.051382582,39.0502391811191,39.048485592,39.0480302785352,39.048157295,39.0617153972115,39.061782676953,39.0621283187262,39.0517863180137]}]],[[{"lng":[-27.94815966,-27.9483804689418,-27.9483969346723,-27.94815966],"lat":[39.050220129,39.0502391811191,39.0517863180137,39.050220129]}]]],null,"Area for development overlay",{"interactive":true,"className":"","stroke":true,"color":"black","weight":0.2,"opacity":0.5,"fill":true,"fillColor":["#EFF3FF","#EFF3FF","#6BAED6","#EFF3FF","#EFF3FF","#C6DBEF","#C6DBEF","#6BAED6","#6BAED6","#3182BD","#6BAED6","#3182BD","#C6DBEF","#C6DBEF","#3182BD","#08519C","#9ECAE1","#9ECAE1"],"fillOpacity":0.6,"smoothFactor":0.2,"noClip":false},["<br>Name Area: Ilhéu da Praia <br>Institution: SPEA <br>Superficie (km²): 1.291","<br>Name Area: Location 1 <br>Institution: Graciosenses <br>Superficie (km²): 0.03","<br>Name Area: Ilhéu da Praia,Zona de mergulho,Location 1,Diving area <br>Institution: SPEA,Clube Navale Da Graciosa,Graciosenses,DivinGraciosa <br>Superficie (km²): 0","<br>Name Area: Location 1 <br>Institution: DRPM <br>Superficie (km²): 1.194","<br>Name Area: Ilhéu de Baixo <br>Institution: SPEA <br>Superficie (km²): 4.669","<br>Name Area: Ilhéu da Praia,Location 1 <br>Institution: SPEA,Graciosenses <br>Superficie (km²): 0.048","<br>Name Area: Reserva Natural do Ilhéu de Baixo,Ilhéu de Baixo <br>Institution: Secretaria Regional do Ambiente e Alteracoes Climaticas,SPEA <br>Superficie (km²): 0.131","<br>Name Area: Reserva Natural do Ilhéu da Praia,Ilhéu da Praia,Zona de mergulho,Location 1 <br>Institution: Secretaria Regional do Ambiente e Alterocoes Clima,SPEA,Clube Navale Da Graciosa,Graciosenses <br>Superficie (km²): 0","<br>Name Area: Location 1,Ilhéu da Praia,Location 1,Diving area <br>Institution: DRPM,SPEA,Graciosenses,DivinGraciosa <br>Superficie (km²): 0.326","<br>Name Area: Location 1,Reserva Natural do Ilhéu da Praia,Ilhéu da Praia,Zona de mergulho,Location 1 <br>Institution: DRPM,Secretaria Regional do Ambiente e Alterocoes Clima,SPEA,Clube Navale Da Graciosa,Graciosenses <br>Superficie (km²): 0","<br>Name Area: Location 1,Ilhéu da Praia,Zona de mergulho,Location 1 <br>Institution: DRPM,SPEA,Clube Navale Da Graciosa,Graciosenses <br>Superficie (km²): 0.258","<br>Name Area: Location 1,Ilhéu da Praia,Zona de mergulho,Location 1,Diving area <br>Institution: DRPM,SPEA,Clube Navale Da Graciosa,Graciosenses,DivinGraciosa <br>Superficie (km²): 0.252","<br>Name Area: Location 1,Ilhéu da Praia <br>Institution: DRPM,SPEA <br>Superficie (km²): 1.126","<br>Name Area: Location 1,Location 1 <br>Institution: DRPM,Graciosenses <br>Superficie (km²): 0.81","<br>Name Area: Location 1,Reserva Natural do Ilhéu da Praia,Ilhéu da Praia,Location 1,Diving area <br>Institution: DRPM,Secretaria Regional do Ambiente e Alterocoes Clima,SPEA,Graciosenses,DivinGraciosa <br>Superficie (km²): 0.148","<br>Name Area: Location 1,Reserva Natural do Ilhéu da Praia,Ilhéu da Praia,Zona de mergulho,Location 1,Diving area <br>Institution: DRPM,Secretaria Regional do Ambiente e Alterocoes Clima,SPEA,Clube Navale Da Graciosa,Graciosenses,DivinGraciosa <br>Superficie (km²): 0.073","<br>Name Area: Location 1,Ilhéu da Praia,Location 1 <br>Institution: DRPM,SPEA,Graciosenses <br>Superficie (km²): 1.065","<br>Name Area: Location 1,Ilhéu da Praia,Zona de mergulho <br>Institution: DRPM,SPEA,Clube Navale Da Graciosa <br>Superficie (km²): 0.002"],null,null,{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},{"color":"lightblue","weight":6}]},{"method":"addPolygons","args":[[[[{"lng":[-27.976661113,-27.941687949,-27.94147399,-27.941153052,-27.961130808,-27.970968766,-27.970443263,-27.976404589,-27.978717528,-27.976661113],"lat":[39.061591899,39.062258242,39.049049607,39.04273517,39.04299699,39.050714114,39.057603858,39.058182082,39.060167286,39.061591899]}]],[[{"lng":[-27.938347376,-27.921640358,-27.920402886,-27.950720953,-27.950468809,-27.938347376],"lat":[39.01571555,39.01307093,38.998406248,38.997684956,39.015748113,39.01571555]}]],[[{"lng":[-27.955052106,-27.941439912,-27.93556192,-27.960930859,-27.969592403,-27.955052106],"lat":[39.067135677,39.062091384,39.045514743,39.048157295,39.064733678,39.067135677]}]],[[{"lng":[-27.972376643,-27.948512821,-27.948361806,-27.968936987,-27.972376643],"lat":[39.061500774,39.062675116,39.048485592,39.04766018,39.061500774]}]],[[{"lng":[-27.961980112,-27.951673081,-27.949518271,-27.960491684,-27.961980112],"lat":[39.059759492,39.060656004,39.052672159,39.052327339,39.059759492]}]],[[{"lng":[-27.956891964,-27.94915764,-27.94815966,-27.961632178,-27.956891964],"lat":[39.056175267,39.056807538,39.050220129,39.051382582,39.056175267]}]],[[{"lng":[-27.954818981,-27.953162232,-27.958753762,-27.956268638,-27.951091294,-27.954818981],"lat":[39.057102923,39.052439219,39.056138044,39.060158287,39.057906979,39.057102923]}]],[[{"lng":[-27.940943806,-27.935973557,-27.938458681,-27.942600556,-27.940943806],"lat":[39.010291088,39.007555418,39.005624293,39.009003727,39.010291088]}]]],null,"Area for development",{"interactive":true,"className":"","stroke":true,"color":"black","weight":1,"opacity":0.5,"fill":true,"fillColor":"black","fillOpacity":0,"smoothFactor":0.2,"noClip":false},["<br>Name Area: Location 1 <br>Institution:  <br>Description:  <br>Superficie (km²): 5.253765","<br>Name Area: Ilhéu de Baixo <br>Institution:  <br>Description:  <br>Superficie (km²): 4.800261","<br>Name Area: Ilhéu da Praia <br>Institution:  <br>Description:  <br>Superficie (km²): 4.58751","<br>Name Area: Location 1 <br>Institution:  <br>Description:  <br>Superficie (km²): 3.009734","<br>Name Area: Diving area <br>Institution:  <br>Description:  <br>Superficie (km²): 0.798972","<br>Name Area: Zona de mergulho <br>Institution:  <br>Description:  <br>Superficie (km²): 0.58455","<br>Name Area: Reserva Natural do Ilhéu da Praia <br>Institution:  <br>Description:  <br>Superficie (km²): 0.220654","<br>Name Area: Reserva Natural do Ilhéu de Baixo <br>Institution:  <br>Description:  <br>Superficie (km²): 0.131388"],null,null,{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},{"color":"lightblue","weight":6}]},{"method":"addLegend","args":[{"colors":["#EFF3FF","#C6DBEF","#9ECAE1","#6BAED6","#3182BD","#08519C"],"labels":["1","2","3","4","5","6"],"na_color":null,"na_label":"NA","opacity":0.5,"position":"topright","type":"factor","title":"Overlay Area for development","extra":null,"layerId":null,"className":"info legend","group":null}]},{"method":"addPolygons","args":[[[[{"lng":[-28.140086641,-28.08647362,-27.989970181,-27.891456253,-27.833822255,-27.830471441,-27.92697488,-28.06971955,-28.167563315,-28.202411779,-28.18096657,-28.140086641],"lat":[39.163243373,39.18973852,39.192335546,39.147133601,39.068090385,38.979581811,38.914952742,38.927466193,38.997813222,39.074854126,39.122182162,39.163243373]}]],[[{"lng":[-27.924810246,-27.945614011,-27.963740063,-27.945614011,-27.925481335,-27.905348659,-27.924810246],"lat":[38.99344746,38.989796288,39.002843909,39.021085936,39.022650055,39.008050265,38.99344746]}]],[[{"lng":[-27.954507536,-27.944757984,-27.92564788,-27.926770691,-27.946532156,-27.950798836,-27.954507536],"lat":[39.008024979,39.021088103,39.014116312,38.998760166,39.000854382,39.002948536,39.008024979]}]],[[{"lng":[-27.953922683,-27.943163011,-27.939128134,-27.949887806,-27.963340929,-27.964346115,-27.953922683],"lat":[39.066877442,39.067138502,39.059045202,39.045989537,39.050166398,39.064266792,39.066877442]}]],[[{"lng":[-27.963570124,-27.967161976,-27.957056681,-27.946726824,-27.944256641,-27.957056681,-27.963570124],"lat":[39.055788548,39.059451024,39.062589628,39.061717808,39.051952681,39.049336793,39.055788548]}]],[[{"lng":[-27.964438616,-27.949265632,-27.948295552,-27.962895514,-27.964438616],"lat":[39.060021376,39.061064011,39.051197037,39.051081988,39.060021376]}]],[[{"lng":[-27.961084281,-27.948449545,-27.948692577,-27.960722271,-27.961084281],"lat":[39.061037727,39.061038211,39.05103598,39.05131911,39.061037727]}]],[[{"lng":[-27.960503017,-27.948732929,-27.949696562,-27.960227683,-27.960503017],"lat":[39.060605956,39.060659402,39.051359223,39.052267913,39.060605956]}]],[[{"lng":[-28.052668863,-28.042873776,-28.038640464,-28.042356444,-28.052819595,-28.052668863],"lat":[39.101309047,39.102882528,39.096975236,39.095665924,39.093042346,39.101309047]}]],[[{"lng":[-27.958823377,-27.95199366,-27.95199366,-27.958031545,-27.958823377],"lat":[39.059321436,39.059812386,39.051276672,39.051757584,39.059321436]}]]],null,"Priority area for conservation",{"interactive":true,"className":"","stroke":true,"color":"black","weight":1,"opacity":0.5,"fill":true,"fillColor":"black","fillOpacity":0,"smoothFactor":0.2,"noClip":false},["<br>Name Area: IBA Graciosa - proteção parcial <br>Institution: SPEA <br>Description: Área abrangente todas as colónias importantes para as aves marinhas <br>Superficie (km²): 745424180","<br>Name Area: Ilhéu de Baixo <br>Institution: SPEA <br>Description: Colónia importante aves marinhas <br>Superficie (km²): 11468134","<br>Name Area: Reserva Natural do Ilhéu de Baixo e áreas adjacentes <br>Institution: Secretaria Regional do Ambiente e Alteracoes Climaticas <br>Description: área fundamental para preservação de recursos naturais. <br>Superficie (km²): 4292657","<br>Name Area: Ilhéu da Praia <br>Institution: SPEA <br>Description: Colónia importante aves marinhas <br>Superficie (km²): 3744215","<br>Name Area: Reserva Natural do Ilhéu da Praia e Áreas adjacentes <br>Institution: Secretaria Regional do Ambiente e Alteracoes Climaticas <br>Description: área fundamental para preservação de recursos naturais. <br>Superficie (km²): 1936087","<br>Name Area: Area protegida do ilhéu da praia <br>Institution: Divingraciosa <br>Description: Zona com potencial para conservação e crescimento ambiental <br>Superficie (km²): 1352117","<br>Name Area: Location 1 <br>Institution: DRPM <br>Description: Inclui a área protegida do PNI e a RN2000, bem como parte da reserva ecológica e da reserva Biosfera <br>Superficie (km²): 1168426","<br>Name Area: ilheu <br>Institution: Clube Navale Da Graciosa <br>Description: com devida fiscalização, que pode ser partilhada com a pesca, clube naval e os atores que tem interesse na região <br>Superficie (km²): 946382","<br>Name Area: Ilhéu da Baleia <br>Institution: SPEA <br>Description: Colónia importante aves marinhas <br>Superficie (km²): 888805","<br>Name Area: Location 1 <br>Institution: Graciosenses <br>Description: reserva e a atividade como é realizada (aqueles que a fazem de forma legal) não perturba o bom desenvolvimento da natureza <br>Superficie (km²): 497698"],null,null,{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},{"color":"lightblue","weight":6}]},{"method":"addPolygons","args":[[[[{"lng":[-27.958031545,-27.9580654379546,-27.958823377,-27.958031545],"lat":[39.051757584,39.0520813411763,39.059321436,39.051757584]}]],[[{"lng":[-28.18096657,-28.140086641,-28.08647362,-27.989970181,-27.891456253,-27.833822255,-27.830471441,-27.92697488,-28.06971955,-28.167563315,-28.202411779,-28.18096657],"lat":[39.122182162,39.163243373,39.18973852,39.192335546,39.147133601,39.068090385,38.979581811,38.914952742,38.927466193,38.997813222,39.074854126,39.122182162]},{"lng":[-27.953922683,-27.964346115,-27.9640712423148,-27.967161976,-27.9637551706197,-27.963340929,-27.949887806,-27.9451185124928,-27.944256641,-27.9444249625664,-27.939128134,-27.943163011,-27.953922683],"lat":[39.066877442,39.064266792,39.0604109751056,39.059451024,39.0559772330582,39.050166398,39.045989537,39.0517765441055,39.051952681,39.0526180897865,39.059045202,39.067138502,39.066877442]},{"lng":[-27.945614011,-27.924810246,-27.905348659,-27.925481335,-27.945614011,-27.963740063,-27.945614011],"lat":[38.989796288,38.99344746,39.008050265,39.022650055,39.021085936,39.002843909,38.989796288]},{"lng":[-28.042356444,-28.038640464,-28.042873776,-28.052668863,-28.052819595,-28.042356444],"lat":[39.095665924,39.096975236,39.102882528,39.101309047,39.093042346,39.095665924]}]],[[{"lng":[-27.9487371059992,-27.9537195800284,-27.9512345864677,-27.9486887389556,-27.9486923291901,-27.9487371059992],"lat":[39.0510370280313,39.0511542951764,39.0511738771468,39.0511939386518,39.0510461788712,39.0510370280313]}],[{"lng":[-27.9496417520584,-27.9556080303334,-27.960503017,-27.9604926671618,-27.9610550834081,-27.961084281,-27.9496417520584],"lat":[39.06103816533,39.0606281833153,39.060605956,39.0602925279611,39.0602538806612,39.061037727,39.06103816533]}],[{"lng":[-27.9492256325591,-27.9492630923869,-27.948449545,-27.9486107625439,-27.9489858578546,-27.948732929,-27.9492256325591],"lat":[39.0606571647156,39.0610381798354,39.061038211,39.0544031377767,39.0582183444377,39.060659402,39.0606571647156]}]],[[{"lng":[-27.9599956105961,-27.960227683,-27.9602348468595,-27.9599956105961],"lat":[39.0522478883632,39.052267913,39.0524848588501,39.0522478883632]}]],[[{"lng":[-27.9486923291901,-27.948692577,-27.9487371059992,-27.9486923291901],"lat":[39.0510461788712,39.05103598,39.0510370280313,39.0510461788712]}]],[[{"lng":[-27.963570124,-27.9602348468595,-27.9607860308001,-27.960722271,-27.9590174434375,-27.9599956105961,-27.9588507404872,-27.962895514,-27.9637373939965,-27.963570124],"lat":[39.055788548,39.0524848588501,39.0530308225969,39.05131911,39.0512789853029,39.0522478883632,39.0511138611753,39.051081988,39.055959106906,39.055788548]}]],[[{"lng":[-27.9487371059992,-27.9487371059992,-27.9486923291901,-27.9487371059992],"lat":[39.0510370280313,39.0510370280313,39.0510461788712,39.0510370280313]}]],[[{"lng":[-27.943163011,-27.939128134,-27.949887806,-27.963340929,-27.9637654755466,-27.9637551706197,-27.9637373939965,-27.962895514,-27.9588507404872,-27.957056681,-27.9487371059992,-27.948692577,-27.9486923291901,-27.9451185124928,-27.9444249625664,-27.946726824,-27.957056681,-27.9640712423148,-27.964346115,-27.953922683,-27.943163011],"lat":[39.067138502,39.059045202,39.045989537,39.050166398,39.0561217869309,39.0559772330582,39.055959106906,39.051081988,39.0511138611753,39.049336793,39.0510370280313,39.05103598,39.0510461788712,39.0517765441055,39.0526180897865,39.061717808,39.062589628,39.0604109751056,39.064266792,39.066877442,39.067138502]}]],[[{"lng":[-27.945614011,-27.925481335,-27.905348659,-27.924810246,-27.945614011,-27.963740063,-27.945614011],"lat":[39.021085936,39.022650055,39.008050265,38.99344746,38.989796288,39.002843909,39.021085936]},{"lng":[-27.946532156,-27.926770691,-27.92564788,-27.944757984,-27.954507536,-27.950798836,-27.946532156],"lat":[39.000854382,38.998760166,39.014116312,39.021088103,39.008024979,39.002948536,39.000854382]}]],[[{"lng":[-28.038640464,-28.042356444,-28.052819595,-28.052668863,-28.042873776,-28.038640464],"lat":[39.096975236,39.095665924,39.093042346,39.101309047,39.102882528,39.096975236]}]],[[{"lng":[-27.9640453949064,-27.9637654755466,-27.964438616,-27.9640453949064],"lat":[39.060048396794,39.0561217869309,39.060021376,39.060048396794]}]],[[{"lng":[-27.9537195800284,-27.9487371059992,-27.957056681,-27.9588507404872,-27.9512345864677,-27.9537195800284],"lat":[39.0511542951764,39.0510370280313,39.049336793,39.0511138611753,39.0511738771468,39.0511542951764]}],[{"lng":[-27.9637654755466,-27.9637373939965,-27.9637551706197,-27.9637654755466],"lat":[39.0561217869309,39.055959106906,39.0559772330582,39.0561217869309]}],[{"lng":[-27.961084281,-27.9610550834081,-27.9640453949064,-27.9640712423148,-27.957056681,-27.946726824,-27.9444249625664,-27.9451185124928,-27.9486923291901,-27.9486887389556,-27.948295552,-27.9489858578546,-27.9486107625439,-27.948449545,-27.9492630923869,-27.949265632,-27.9556080303334,-27.9496417520584,-27.961084281],"lat":[39.061037727,39.0602538806612,39.060048396794,39.0604109751056,39.062589628,39.061717808,39.0526180897865,39.0517765441055,39.0510461788712,39.0511939386518,39.051197037,39.0582183444377,39.0544031377767,39.061038211,39.0610381798354,39.061064011,39.0606281833153,39.06103816533,39.061037727]}]],[[{"lng":[-27.944256641,-27.9451185124928,-27.9444249625664,-27.944256641],"lat":[39.051952681,39.0517765441055,39.0526180897865,39.051952681]}],[{"lng":[-27.9640453949064,-27.964438616,-27.9637654755466,-27.9637551706197,-27.967161976,-27.9640712423148,-27.9640453949064],"lat":[39.060048396794,39.060021376,39.0561217869309,39.0559772330582,39.059451024,39.0604109751056,39.060048396794]}]],[[{"lng":[-27.9599956105961,-27.9580654379546,-27.958031545,-27.95199366,-27.95199366,-27.949696562,-27.9489858578546,-27.9486107625439,-27.9486887389556,-27.9537195800284,-27.9590174434375,-27.9599956105961],"lat":[39.0522478883632,39.0520813411763,39.051757584,39.051276672,39.0515574307674,39.051359223,39.0582183444377,39.0544031377767,39.0511939386518,39.0511542951764,39.0512789853029,39.0522478883632]}],[{"lng":[-27.9610550834081,-27.9604926671618,-27.9602348468595,-27.9607860308001,-27.9610550834081],"lat":[39.0602538806612,39.0602925279611,39.0524848588501,39.0530308225969,39.0602538806612]}],[{"lng":[-27.9492630923869,-27.9492256325591,-27.9556080303334,-27.9496417520584,-27.9492630923869],"lat":[39.0610381798354,39.0606571647156,39.0606281833153,39.06103816533,39.0610381798354]}]],[[{"lng":[-27.948732929,-27.9489858578546,-27.9492256325591,-27.948732929],"lat":[39.060659402,39.0582183444377,39.0606571647156,39.060659402]}],[{"lng":[-27.960503017,-27.9556080303334,-27.9604926671618,-27.960503017],"lat":[39.060605956,39.0606281833153,39.0602925279611,39.060605956]}]],[[{"lng":[-27.9590174434375,-27.9537195800284,-27.9588507404872,-27.9590174434375],"lat":[39.0512789853029,39.0511542951764,39.0511138611753,39.0512789853029]}],[{"lng":[-27.9607860308001,-27.963570124,-27.9637373939965,-27.9637654755466,-27.9640453949064,-27.9610550834081,-27.9607860308001],"lat":[39.0530308225969,39.055788548,39.055959106906,39.0561217869309,39.060048396794,39.0602538806612,39.0530308225969]}],[{"lng":[-27.9486887389556,-27.9486107625439,-27.948295552,-27.9486887389556],"lat":[39.0511939386518,39.0544031377767,39.051197037,39.0511939386518]}],[{"lng":[-27.9496417520584,-27.949265632,-27.9492630923869,-27.9496417520584],"lat":[39.06103816533,39.061064011,39.0610381798354,39.06103816533]}]],[[{"lng":[-27.92564788,-27.926770691,-27.946532156,-27.950798836,-27.954507536,-27.944757984,-27.92564788],"lat":[39.014116312,38.998760166,39.000854382,39.002948536,39.008024979,39.021088103,39.014116312]}]],[[{"lng":[-27.95199366,-27.9580654379546,-27.958823377,-27.95199366,-27.95199366],"lat":[39.0515574307674,39.0520813411763,39.059321436,39.059812386,39.0515574307674]}]],[[{"lng":[-27.95199366,-27.958031545,-27.9580654379546,-27.95199366,-27.95199366],"lat":[39.051276672,39.051757584,39.0520813411763,39.0515574307674,39.051276672]}]],[[{"lng":[-27.9607860308001,-27.9602348468595,-27.960227683,-27.9599956105961,-27.9590174434375,-27.960722271,-27.9607860308001],"lat":[39.0530308225969,39.0524848588501,39.052267913,39.0522478883632,39.0512789853029,39.05131911,39.0530308225969]}]],[[{"lng":[-27.95199366,-27.958823377,-27.9580654379546,-27.9599956105961,-27.9602348468595,-27.9604926671618,-27.9556080303334,-27.9492256325591,-27.9489858578546,-27.949696562,-27.95199366,-27.95199366],"lat":[39.059812386,39.059321436,39.0520813411763,39.0522478883632,39.0524848588501,39.0602925279611,39.0606281833153,39.0606571647156,39.0582183444377,39.051359223,39.0515574307674,39.059812386]}]]],null,"Priority areas for conservation overlay",{"interactive":true,"className":"","stroke":true,"color":"black","weight":0.2,"opacity":0.5,"fill":true,"fillColor":["#EDF8E9","#EDF8E9","#74C476","#41AB5D","#A1D99B","#A1D99B","#EDF8E9","#C7E9C0","#C7E9C0","#C7E9C0","#A1D99B","#A1D99B","#C7E9C0","#41AB5D","#41AB5D","#74C476","#A1D99B","#005A32","#238B45","#74C476","#238B45"],"fillOpacity":0.8,"smoothFactor":0.2,"noClip":false},["<br>Name Area: Area protegida do ilhéu da praia <br>Institution: Divingraciosa <br>Superficie (km²): 0","<br>Name Area: IBA Graciosa - proteção parcial <br>Institution: SPEA <br>Superficie (km²): 72.925","<br>Name Area: Location 1,Reserva Natural do Ilhéu da Praia e Áreas adjacentes,IBA Graciosa - proteção parcial,Ilhéu da Praia <br>Institution: DRPM,Secretaria Regional do Ambiente e Alteracoes Climaticas,SPEA,SPEA <br>Superficie (km²): 0.006","<br>Name Area: Location 1,IBA Graciosa - proteção parcial,Ilhéu da Praia,ilheu,Area protegida do ilhéu da praia <br>Institution: DRPM,SPEA,SPEA,Clube Navale Da Graciosa,Divingraciosa <br>Superficie (km²): 0","<br>Name Area: Location 1,IBA Graciosa - proteção parcial,Ilhéu da Praia <br>Institution: DRPM,SPEA,SPEA <br>Superficie (km²): 0","<br>Name Area: IBA Graciosa - proteção parcial,Ilhéu da Praia,Area protegida do ilhéu da praia <br>Institution: SPEA,SPEA,Divingraciosa <br>Superficie (km²): 0.008","<br>Name Area: Location 1 <br>Institution: DRPM <br>Superficie (km²): 0","<br>Name Area: IBA Graciosa - proteção parcial,Ilhéu da Praia <br>Institution: SPEA,SPEA <br>Superficie (km²): 0.178","<br>Name Area: IBA Graciosa - proteção parcial,Ilhéu de Baixo <br>Institution: SPEA,SPEA <br>Superficie (km²): 0.718","<br>Name Area: IBA Graciosa - proteção parcial,Ilhéu da Baleia <br>Institution: SPEA,SPEA <br>Superficie (km²): 0.089","<br>Name Area: Reserva Natural do Ilhéu da Praia e Áreas adjacentes,IBA Graciosa - proteção parcial,Area protegida do ilhéu da praia <br>Institution: Secretaria Regional do Ambiente e Alteracoes Climaticas,SPEA,Divingraciosa <br>Superficie (km²): 0.001","<br>Name Area: Reserva Natural do Ilhéu da Praia e Áreas adjacentes,IBA Graciosa - proteção parcial,Ilhéu da Praia <br>Institution: Secretaria Regional do Ambiente e Alteracoes Climaticas,SPEA,SPEA <br>Superficie (km²): 0.054","<br>Name Area: Reserva Natural do Ilhéu da Praia e Áreas adjacentes,IBA Graciosa - proteção parcial <br>Institution: Secretaria Regional do Ambiente e Alteracoes Climaticas,SPEA <br>Superficie (km²): 0.006","<br>Name Area: Location 1,Reserva Natural do Ilhéu da Praia e Áreas adjacentes,IBA Graciosa - proteção parcial,Ilhéu da Praia,Area protegida do ilhéu da praia <br>Institution: DRPM,Secretaria Regional do Ambiente e Alteracoes Climaticas,SPEA,SPEA,Divingraciosa <br>Superficie (km²): 0.013","<br>Name Area: Location 1,Reserva Natural do Ilhéu da Praia e Áreas adjacentes,IBA Graciosa - proteção parcial,Ilhéu da Praia,ilheu <br>Institution: DRPM,Secretaria Regional do Ambiente e Alteracoes Climaticas,SPEA,SPEA,Clube Navale Da Graciosa <br>Superficie (km²): 0.001","<br>Name Area: Reserva Natural do Ilhéu da Praia e Áreas adjacentes,IBA Graciosa - proteção parcial,Ilhéu da Praia,Area protegida do ilhéu da praia <br>Institution: Secretaria Regional do Ambiente e Alteracoes Climaticas,SPEA,SPEA,Divingraciosa <br>Superficie (km²): 0.017","<br>Name Area: Reserva Natural do Ilhéu de Baixo e áreas adjacentes,IBA Graciosa - proteção parcial,Ilhéu de Baixo <br>Institution: Secretaria Regional do Ambiente e Alteracoes Climaticas,SPEA,SPEA <br>Superficie (km²): 0.429","<br>Name Area: Location 1,Reserva Natural do Ilhéu da Praia e Áreas adjacentes,IBA Graciosa - proteção parcial,Ilhéu da Praia,ilheu,Location 1,Area protegida do ilhéu da praia <br>Institution: DRPM,Secretaria Regional do Ambiente e Alteracoes Climaticas,SPEA,SPEA,Clube Navale Da Graciosa,Graciosenses,Divingraciosa <br>Superficie (km²): 0.048","<br>Name Area: Location 1,Reserva Natural do Ilhéu da Praia e Áreas adjacentes,IBA Graciosa - proteção parcial,Ilhéu da Praia,Location 1,Area protegida do ilhéu da praia <br>Institution: DRPM,Secretaria Regional do Ambiente e Alteracoes Climaticas,SPEA,SPEA,Graciosenses,Divingraciosa <br>Superficie (km²): 0.002","<br>Name Area: Location 1,IBA Graciosa - proteção parcial,Ilhéu da Praia,Area protegida do ilhéu da praia <br>Institution: DRPM,SPEA,SPEA,Divingraciosa <br>Superficie (km²): 0.001","<br>Name Area: Location 1,Reserva Natural do Ilhéu da Praia e Áreas adjacentes,IBA Graciosa - proteção parcial,Ilhéu da Praia,ilheu,Area protegida do ilhéu da praia <br>Institution: DRPM,Secretaria Regional do Ambiente e Alteracoes Climaticas,SPEA,SPEA,Clube Navale Da Graciosa,Divingraciosa <br>Superficie (km²): 0.045"],null,null,{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},{"color":"lightblue","weight":6}]},{"method":"addLegend","args":[{"colors":["#EDF8E9","#C7E9C0","#A1D99B","#74C476","#41AB5D","#238B45","#005A32"],"labels":["1","2","3","4","5","6","7"],"na_color":null,"na_label":"NA","opacity":0.5,"position":"topright","type":"factor","title":"Overlay Priority area for conservation","extra":null,"layerId":null,"className":"info legend","group":null}]},{"method":"addPolygons","args":[[[[{"lng":[-27.9558718438502,-27.955654933,-27.9558718438502,-27.9558718438502],"lat":[39.0602095313445,39.059345479,39.0602095313445,39.0602095313445]}]],[[{"lng":[-27.868647846,-28.037990512,-28.156598938,-28.203037608,-28.183676617,-28.113403388,-28.088039154,-28.060615241,-27.974229913,-27.889901379,-27.829568769,-27.824769584,-27.868647846],"lat":[38.936078792,38.912610123,38.977663259,39.070527713,39.123952122,39.181227037,39.190533265,39.195315449,39.185750755,39.142693527,39.063953004,39.000577594,38.936078792]},{"lng":[-27.948651938,-27.975874974,-27.969594945,-27.946440676,-27.948651938],"lat":[39.06228626,39.061093321,39.049074744,39.048813451,39.06228626]},{"lng":[-28.067744566,-28.063061292,-28.067658901,-28.072025567,-28.070610417,-28.067744566],"lat":[39.060501159,39.063396807,39.068230246,39.067158672,39.063833956,39.060501159]}],[{"lng":[-27.95598866,-27.9558718438502,-27.9558718438502,-27.95598866],"lat":[39.060674862,39.0602095313445,39.0602095313445,39.060674862]}]],[[{"lng":[-27.957804828,-27.9577307888831,-27.957611561,-27.957804828],"lat":[39.060595112,39.0601047004914,39.059314973,39.060595112]}]],[[{"lng":[-27.9558718438502,-27.955654933,-27.957611561,-27.9577307888831,-27.9558718438502],"lat":[39.0602095313445,39.059345479,39.059314973,39.0601047004914,39.0602095313445]}]],[[{"lng":[-27.9558718438502,-27.9577307888831,-27.957804828,-27.95598866,-27.9558718438502],"lat":[39.0602095313445,39.0601047004914,39.060595112,39.060674862,39.0602095313445]}]],[[{"lng":[-27.949234674,-27.949365499,-27.959766078,-27.960943503,-27.9577307888831,-27.957611561,-27.955654933,-27.9558718438502,-27.949234674],"lat":[39.060583819,39.051491562,39.051745551,39.059923527,39.0601047004914,39.059314973,39.059345479,39.0602095313445,39.060583819]}]],[[{"lng":[-27.948651938,-27.946440676,-27.969594945,-27.975874974,-27.948651938],"lat":[39.06228626,39.048813451,39.049074744,39.061093321,39.06228626]},{"lng":[-27.949365499,-27.949234674,-27.9558718438502,-27.95598866,-27.957804828,-27.9577307888831,-27.960943503,-27.959766078,-27.949365499],"lat":[39.051491562,39.060583819,39.0602095313445,39.060674862,39.060595112,39.0601047004914,39.059923527,39.051745551,39.051491562]},{"lng":[-27.965802921,-27.965805517,-27.966341585,-27.966372886,-27.965802921],"lat":[39.052479008,39.052826164,39.052898005,39.05244739,39.052479008]}]],[[{"lng":[-28.063061292,-28.067744566,-28.070610417,-28.072025567,-28.067658901,-28.063061292],"lat":[39.063396807,39.060501159,39.063833956,39.067158672,39.068230246,39.063396807]}]],[[{"lng":[-27.965802921,-27.966372886,-27.966341585,-27.965805517,-27.965802921],"lat":[39.052479008,39.05244739,39.052898005,39.052826164,39.052479008]}]]],null,"Overlay area for expansion",{"interactive":true,"className":"","stroke":true,"color":"black","weight":0.2,"opacity":0.5,"fill":true,"fillColor":["#FDBE85","#FEEDDE","#FDBE85","#D94701","#FD8D3C","#FD8D3C","#FDBE85","#FDBE85","#FD8D3C"],"fillOpacity":0.6,"smoothFactor":0.2,"noClip":false},["<br>Name Area: Área importante para as aves marinhas Graciosa - IBA Graciosa,Zona de expansão <br>Institution: SPEA,Clube Navale Da Graciosa <br>Superficie (km²): 0","<br>Name Area: Área importante para as aves marinhas Graciosa - IBA Graciosa <br>Institution: SPEA <br>Superficie (km²): 754.683","<br>Name Area: Location 1,Zona a norte do ilheu <br>Institution: Graciosenses,DivinGraciosa <br>Superficie (km²): 0","<br>Name Area: Área importante para as aves marinhas Graciosa - IBA Graciosa,Zona de expansão,Location 1,Zona a norte do ilheu <br>Institution: SPEA,Clube Navale Da Graciosa,Graciosenses,DivinGraciosa <br>Superficie (km²): 0.015","<br>Name Area: Área importante para as aves marinhas Graciosa - IBA Graciosa,Location 1,Zona a norte do ilheu <br>Institution: SPEA,Graciosenses,DivinGraciosa <br>Superficie (km²): 0.009","<br>Name Area: Área importante para as aves marinhas Graciosa - IBA Graciosa,Zona de expansão,Location 1 <br>Institution: SPEA,Clube Navale Da Graciosa,Graciosenses <br>Superficie (km²): 0.903","<br>Name Area: Área importante para as aves marinhas Graciosa - IBA Graciosa,Location 1 <br>Institution: SPEA,Graciosenses <br>Superficie (km²): 2.175","<br>Name Area: Porto Afonso,Área importante para as aves marinhas Graciosa - IBA Graciosa <br>Institution: Secretaria Regional do Ambiente e Alteracoes Climaticas,SPEA <br>Superficie (km²): 0.358","<br>Name Area: Location 1,Área importante para as aves marinhas Graciosa - IBA Graciosa,Location 1 <br>Institution: DRPM,SPEA,Graciosenses <br>Superficie (km²): 0.002"],null,null,{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},{"color":"lightblue","weight":6}]},{"method":"addPolygons","args":[[[[{"lng":[-28.113403388,-28.088039154,-28.060615241,-27.974229913,-27.889901379,-27.829568769,-27.824769584,-27.868647846,-28.037990512,-28.156598938,-28.203037608,-28.183676617,-28.113403388],"lat":[39.181227037,39.190533265,39.195315449,39.185750755,39.142693527,39.063953004,39.000577594,38.936078792,38.912610123,38.977663259,39.070527713,39.123952122,39.181227037]}]],[[{"lng":[-27.975874974,-27.948651938,-27.946440676,-27.969594945,-27.975874974],"lat":[39.061093321,39.06228626,39.048813451,39.049074744,39.061093321]}]],[[{"lng":[-27.960943503,-27.949234674,-27.949365499,-27.959766078,-27.960943503],"lat":[39.059923527,39.060583819,39.051491562,39.051745551,39.059923527]}]],[[{"lng":[-28.072025567,-28.067658901,-28.063061292,-28.067744566,-28.070610417,-28.072025567],"lat":[39.067158672,39.068230246,39.063396807,39.060501159,39.063833956,39.067158672]}]],[[{"lng":[-27.957804828,-27.95598866,-27.955654933,-27.957611561,-27.957804828],"lat":[39.060595112,39.060674862,39.059345479,39.059314973,39.060595112]}]],[[{"lng":[-27.966341585,-27.965805517,-27.965802921,-27.966372886,-27.966341585],"lat":[39.052898005,39.052826164,39.052479008,39.05244739,39.052898005]}]]],null,"Area for expansion",{"interactive":true,"className":"","stroke":true,"color":"black","weight":1,"opacity":0.5,"fill":true,"fillColor":"black","fillOpacity":0,"smoothFactor":0.2,"noClip":false},["<br>Name Area: Área importante para as aves marinhas Graciosa - IBA Graciosa <br>Institution: SPEA <br>Description: Área envolvente os 3 principais ilhéus/colónias de aves marinhas. A mesma não implica restrição total das outras atividades, mas proteção parcial nas áreas circundantes e totais nos ilhéus. Ex: Pesca linha de mão deveria ser apenas restrita na <br>Superficie (km²): 758.145","<br>Name Area: Location 1 <br>Institution: Graciosenses <br>Description: se as diversas áreas de atividade pudessem coexistir melhor <br>Superficie (km²): 3.104","<br>Name Area: Zona de expansão <br>Institution: Clube Navale Da Graciosa <br>Description: Consideram não ter uma zona proibitiva, e deve ser trabalhado de base com a educação ambiental <br>Superficie (km²): 0.918","<br>Name Area: Porto Afonso <br>Institution: Secretaria Regional do Ambiente e Alteracoes Climaticas <br>Description: Área de grande potencial geológico <br>Superficie (km²): 0.358","<br>Name Area: Zona a norte do ilheu <br>Institution: DivinGraciosa <br>Description: Zona com potencial para o mergulho <br>Superficie (km²): 0.024","<br>Name Area: Location 1 <br>Institution: DRPM <br>Description: Não se aplica. <br>Superficie (km²): 0.002"],null,null,{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},{"color":"lightblue","weight":6}]},{"method":"addLegend","args":[{"colors":["#FEEDDE","#FDBE85","#FD8D3C","#D94701"],"labels":["1","2","3","4"],"na_color":null,"na_label":"NA","opacity":0.5,"position":"topright","type":"factor","title":"Overlay area for expansion","extra":null,"layerId":null,"className":"info legend","group":null}]},{"method":"addLayersControl","args":["Esri.WorldImagery",["Conflicts","Conflicts overlay","Area for development","Area for development overlay","Priority area for conservation","Priority areas for conservation overlay","Area for expansion","Overlay area for expansion"],{"collapsed":false,"autoZIndex":true,"position":"topleft"}]},{"method":"hideGroup","args":[["Conflicts","Conflicts overlay","Area for development","Area for development overlay","Priority area for conservation","Priority areas for conservation overlay"]]},{"method":"addScaleBar","args":[{"maxWidth":100,"metric":true,"imperial":true,"updateWhenIdle":true,"position":"bottomleft"}]}],"limits":{"lat":[38.912610123,39.195315449],"lng":[-28.203037608,-27.824769584]}},"evals":[],"jsHooks":{"render":[{"code":"function(el, x, data) {\n  return (function(el, x, data) {\n                     // we need a new div element because we have to handle\n                     // the mouseover output seperately\n                     // debugger;\n                     function addElement () {\n                     // generate new div Element\n                     var newDiv = $(document.createElement('div'));\n                     // append at end of leaflet htmlwidget container\n                     $(el).append(newDiv);\n                     //provide ID and style\n                     newDiv.addClass('logo');\nnewDiv.css({\n                           'position': 'absolute',\n                           'bottom': '40px',\n                           'left': '5px',\n                           'background-color': 'transparent',\n                           'border': '0px solid black',\n                           'width': '100px',\n                           'height': '100px',\n                           });return newDiv;\n                     }// check for already existing logo class to not duplicate\n                    var logo = $(el).find('.logo');\n                    if(!logo.length) {\n                    logo = addElement();logo.html('<img src=\"https://msp4bio.eu/wp-content/uploads/2022/10/MSP4BIO.png\", width=100, height=100, style=\"opacity:1;filter:alpha(opacity=100);\"><\/a>');\n                       var map = HTMLWidgets.find('#' + el.id).getMap();\n                       };\n                       }).call(this.getMap(), el, x, data);\n}","data":null},{"code":"function(el, x, data) {\n  return (\n    function(el, x) {\n      document.title = 'MSP4BIO Graciosa';\n    }\n  ).call(this.getMap(), el, x, data);\n}","data":null}]}}</script>
<script type="application/htmlwidget-sizing" data-for="htmlwidget-8f7a3f3bb4e8c93e365c">{"viewer":{"width":"100%","height":400,"padding":0,"fill":true},"browser":{"width":"100%","height":400,"padding":0,"fill":true}}</script>
</body>
</html>
